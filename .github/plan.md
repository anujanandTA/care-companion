# Plan: Care Companion — Full Implementation Plan (4 Phases)

## TL;DR
Build three AI features for home care (Pre-Visit Brief, Discharge Summary, Escalation Agent) across 4 phases. The codebase is fully scaffolded but 100% empty — every `.py` file is a stub. We start with shared infrastructure (config, dependencies, data models), then build the data foundation, then implement each feature progressively from simple RAG → multi-document reasoning → stateful agentic workflows.

---

## Phase 0: Shared Infrastructure (Pre-requisite for all phases)

**Goal:** Set up configuration, dependencies, utilities, and data models that all features depend on.

### Step 0.1 — `requirements.txt`
Populate with all dependencies:
- `langchain`, `langchain-openai`, `langchain-community`, `langgraph`
- `chromadb`
- `neo4j` (Python driver)
- `openai`
- `streamlit`
- `python-dotenv`
- `pydantic` (for data models/schemas)
- `fpdf2` or `reportlab` (for generating synthetic referral PDFs)
- `pypdf` or `pdfplumber` (for parsing referral PDFs)
- `pytest`, `pytest-asyncio`

### Step 0.2 — `src/care_companion/config/settings.py`
- Define a Pydantic `Settings` class using `pydantic-settings` / `BaseSettings`
- Fields: `OPENAI_API_KEY`, `NEO4J_URI`, `NEO4J_USER`, `NEO4J_PASSWORD`, `CHROMA_PERSIST_DIR`, `DATA_DIR`, `CHECKPOINT_DIR`
- Load from `.env` via `python-dotenv`

### Step 0.3 — `src/care_companion/utils/env.py`
- Helper to load `.env` and expose a `get_settings()` singleton

### Step 0.4 — `src/care_companion/utils/logging.py`
- Configure structured logging (stdlib `logging`) with a project-wide logger factory

### Step 0.5 — `src/care_companion/data/models.py`
Define Pydantic models:
- `Patient` — id, name, age, conditions, medications, allergies
- `ProgressNote` — id, patient_id, date, nurse_name, visit_type, subjective, objective, assessment, plan, vitals
- `Referral` — id, patient_id, referral_date, diagnosis, requested_services (list), physician, goals, frequency
- `IncidentReport` — id, patient_id, incident_type, severity, description, actions_taken, reported_by, timestamp

### Step 0.6 — `src/care_companion/data/schemas.py`
Define response/output schemas:
- `RiskProfile` — risk_level, key_findings, medications_of_concern, recent_changes, citations
- `ComplianceGap` — service, status (delivered/missing/partial), evidence, note_reference
- `DischargeReport` — patient_summary, services_delivered, compliance_gaps, recommendations, citations
- `EscalationRecord` — incident_type, severity, follow_up_actions, notifications, structured fields from agent

### Step 0.7 — Create `.env.example`
Template with placeholder values for all required env vars.

### Step 0.8 — `app.py` (Streamlit entry point)
- Minimal Streamlit multipage app setup with sidebar navigation
- Title, description, and links to the 3 feature pages

**Files:** `requirements.txt`, `settings.py`, `env.py`, `logging.py`, `models.py`, `schemas.py`, `.env.example`, `app.py`
**Depends on:** Nothing
**Verification:** `pip install -r requirements.txt` succeeds; `python -c "from src.care_companion.config.settings import Settings"` imports cleanly; `streamlit run app.py` shows the landing page.

---

## Phase I: The Data Foundation (Weeks 1–2)

**Goal:** Generate synthetic Referral PDFs and matching Progress Notes. Build ingestion pipeline to parse and chunk them.

### Step 1.1 — `scripts/generate_synthetic_data.py` (*depends on Step 0.5*)
- Use GPT-4o-mini to generate 3–5 synthetic patient personas
- For each patient, generate:
  - 1 Referral document (structured JSON → rendered to PDF via `fpdf2`)
  - 5 Progress Notes (structured JSON → saved as `.json` or `.md` in `data/progress_notes/`)
- Notes should span 4–6 weeks with evolving clinical data (symptoms improving, new issues, med changes)
- Save referrals to `data/referrals/` as PDFs
- Ensure notes reference the referral's requested services (some fulfilled, some deliberately missing for compliance testing in Phase III)
- Include medical synonym variance (e.g., "SOB", "shortness of breath", "dyspnea") to test normalization later

### Step 1.2 — `src/care_companion/ingestion/note_parser.py` (*depends on Step 0.5*)
- Parse progress note JSON/markdown files into `ProgressNote` Pydantic models
- Extract structured fields: SOAP sections, vitals, date, nurse

### Step 1.3 — `src/care_companion/ingestion/referral_parser.py` (*depends on Step 0.5*)
- Parse referral PDFs using `pypdf` / `pdfplumber`
- Extract into `Referral` Pydantic model: diagnosis, requested services list, goals, frequency

### Step 1.4 — `src/care_companion/ingestion/chunking.py`
- Implement chunking strategy for progress notes (by SOAP section, with metadata preserved)
- Each chunk carries: `patient_id`, `note_id`, `date`, `section_type` as metadata

### Step 1.5 — `src/care_companion/ingestion/loaders.py` (*depends on Steps 1.2–1.4*)
- Orchestrator that loads all notes + referrals for a patient
- Returns structured data ready for vector store ingestion and graph insertion

### Step 1.6 — `scripts/load_sample_data.py` (*depends on Steps 1.1–1.5*)
- Script that runs the full pipeline: generate data → parse → chunk → load into ChromaDB
- Populates `data/vectorstore/` with embeddings

### Step 1.7 — Write tests for Phase I
- `tests/` — test note parsing, referral parsing, chunking (edge cases: missing fields, malformed data)

**Files:** `generate_synthetic_data.py`, `note_parser.py`, `referral_parser.py`, `chunking.py`, `loaders.py`, `load_sample_data.py`
**Depends on:** Phase 0
**Verification:** Run `python scripts/generate_synthetic_data.py` → verify PDFs and note files exist in `data/`; `python scripts/load_sample_data.py` → verify ChromaDB collection is populated; `pytest tests/` passes.

---

## Phase II: Pre-Visit Brief — RAG Pipeline (Weeks 3–4)

**Goal:** Build the RAG pipeline that summarizes progress notes into a risk profile for nurses.

### Step 2.1 — `src/care_companion/retrieval/chroma_store.py` (*depends on Phase I*)
- ChromaDB client wrapper: `initialize_store()`, `add_documents()`, `search(query, patient_id, k)`
- Use OpenAI embeddings (`text-embedding-3-small`)
- Filter by `patient_id` metadata on retrieval

### Step 2.2 — `src/care_companion/graph/neo4j_client.py` (*parallel with Step 2.1*)
- Async Neo4j driver wrapper: `connect()`, `close()`, `execute_query(cypher, params)`
- All queries parameterized — never interpolate user input
- Connection pooling, error handling

### Step 2.3 — `src/care_companion/graph/normalization.py` (*depends on Step 2.2*)
- LLM-based entity normalization: given a medical term, return the canonical form
- Use GPT-4o-mini for normalization calls
- Maintain a local cache/dictionary of known synonyms to reduce API calls
- E.g., {"SOB": "Shortness of Breath", "HF": "Heart Failure", "BP": "Blood Pressure"}

### Step 2.4 — `src/care_companion/graph/triplets.py` (*depends on Step 2.3*)
- Extract Subject-Predicate-Object triplets from progress note text using LLM
- Normalize entities before returning triplets
- Output: list of `(subject, predicate, object)` with labels

### Step 2.5 — `src/care_companion/graph/queries.py` (*depends on Step 2.2*)
- Cypher query templates (parameterized) for:
  - Inserting patients, symptoms, medications, conditions, clinical events
  - Querying patient's full clinical graph
  - Finding related symptoms/conditions for a patient
  - Deduplication: MERGE instead of CREATE for nodes

### Step 2.6 — `src/care_companion/retrieval/hybrid.py` (*depends on Steps 2.1, 2.5*)
- Hybrid retrieval: combine ChromaDB vector search results with Neo4j graph traversal
- Vector search: semantic similarity on note chunks
- Graph search: structured relationships (patient → symptoms → conditions)
- Merge and deduplicate results, rank by relevance

### Step 2.7 — `src/care_companion/retrieval/citation_builder.py` (*depends on Step 2.6*)
- For each retrieved chunk/graph node, build a citation reference
- Format: `[Note 3, 2024-01-15, Assessment]` or `[Graph: Patient → HAS_SYMPTOM → Shortness of Breath]`
- Every AI claim must map back to a source

### Step 2.8 — `src/care_companion/features/pre_visit_brief/prompts.py`
- Define prompt templates for:
  - Risk profile generation (system prompt + retrieved context + output schema)
  - Key findings extraction
  - Medication concerns identification

### Step 2.9 — `src/care_companion/features/pre_visit_brief/chain.py` (*depends on Steps 2.6–2.8*)
- LangChain chain: retrieval → prompt → LLM (GPT-4o) → structured output (`RiskProfile`)
- Input: patient_id
- Pipeline: hybrid retrieval → format context with citations → LLM synthesis → parse to schema

### Step 2.10 — `src/care_companion/features/pre_visit_brief/service.py` (*depends on Step 2.9*)
- Service layer: `generate_pre_visit_brief(patient_id) -> RiskProfile`
- Orchestrates the chain, handles errors, returns structured result

### Step 2.11 — `pages/01_Pre_Visit_Brief.py` (*depends on Step 2.10*)
- Streamlit page:
  - Patient selector (dropdown or search)
  - "Generate Brief" button
  - Display: risk level badge, key findings, medication concerns, recent changes
  - Each finding shows its citation/source
  - Loading state while LLM processes

### Step 2.12 — Write tests for Phase II
- `tests/retrieval/test_citation_builder.py` — citation formatting
- `tests/graph/test_normalization.py` — synonym resolution, cache behavior
- `tests/features/test_pre_visit_brief.py` — chain with mocked LLM, edge cases (no notes, single note)

**Files:** `chroma_store.py`, `neo4j_client.py`, `normalization.py`, `triplets.py`, `queries.py`, `hybrid.py`, `citation_builder.py`, `prompts.py` (pre_visit), `chain.py` (pre_visit), `service.py`, `01_Pre_Visit_Brief.py`
**Depends on:** Phase I (data must exist)
**Verification:** `pytest tests/` passes; Streamlit page loads, selecting a patient and clicking "Generate Brief" produces a risk profile with citations; Neo4j browser shows populated graph; manual review of 2–3 briefs for groundedness.

---

## Phase III: Discharge Summary & Compliance (Weeks 5–6)

**Goal:** Multi-document RAG that cross-references progress notes against the referral to find compliance gaps.

### Step 3.1 — `src/care_companion/features/discharge_summary/prompts.py`
- Prompt templates for:
  - Service-by-service compliance check (referral services vs. notes evidence)
  - Discharge narrative generation
  - Gap identification with severity rating

### Step 3.2 — `src/care_companion/features/discharge_summary/compliance.py` (*depends on Phase II retrieval layer*)
- Core compliance logic:
  - Extract requested services from `Referral`
  - For each service, search notes (hybrid retrieval) for evidence of delivery
  - Classify each service: `delivered` / `partially_delivered` / `missing`
  - Return list of `ComplianceGap` objects with evidence citations

### Step 3.3 — `src/care_companion/features/discharge_summary/chain.py` (*depends on Steps 3.1, 3.2*)
- LangChain chain for discharge report generation:
  - Input: patient_id
  - Step 1: Retrieve referral + all notes via hybrid retrieval
  - Step 2: Run compliance check (Step 3.2)
  - Step 3: LLM synthesis (GPT-4o) → `DischargeReport` with compliance gaps highlighted
  - Every claim cited

### Step 3.4 — `pages/02_Discharge_Summary.py` (*depends on Step 3.3*)
- Streamlit page:
  - Patient selector
  - "Generate Discharge Summary" button
  - Display: patient summary narrative, services table (color-coded: green/yellow/red), compliance gaps detail, recommendations
  - Expandable sections for each gap showing source citations
  - Download button for report (optional: PDF export)

### Step 3.5 — Write tests for Phase III
- `tests/features/test_discharge_summary.py` — compliance logic with mocked data:
  - All services delivered → no gaps
  - Some services missing → correct gaps identified
  - Edge: referral with no matching notes

**Files:** `prompts.py` (discharge), `compliance.py`, `chain.py` (discharge), `02_Discharge_Summary.py`
**Depends on:** Phase II (retrieval layer, graph, citations)
**Verification:** `pytest tests/` passes; Streamlit page generates a discharge report with color-coded compliance table; verify at least one patient shows gaps (by design from synthetic data); manual review of compliance accuracy.

---

## Phase IV: Escalation Agent — Stateful Workflow (Weeks 7–8)

**Goal:** Build a conversational agent using LangGraph that follows internal procedures to record and escalate incidents.

### Step 4.1 — Create escalation procedure documents in `data/procedures/`
- Write 2–3 synthetic procedure documents (JSON or Markdown):
  - Fall incident procedure (required questions: injury assessment, witness, location, time, immediate actions)
  - Medication error procedure (required questions: drug, dose, intended vs. actual, patient status)
  - General incident procedure (fallback)
- Each procedure defines: required fields, severity classification rules, notification chain, documentation requirements

### Step 4.2 — `src/care_companion/features/escalation_agent/state.py` (*depends on Step 0.6*)
- Define LangGraph `TypedDict` state:
  - `messages`: list of chat messages
  - `incident_type`: str (detected or None)
  - `procedure`: loaded procedure document
  - `collected_fields`: dict of answered questions
  - `missing_fields`: list of remaining required questions
  - `severity`: str (assessed level)
  - `escalation_status`: enum (assess → notify → document → resolve)
  - `incident_report`: final `IncidentReport` or None
- State must be fully serializable for LangGraph checkpoints

### Step 4.3 — `src/care_companion/features/escalation_agent/prompts.py`
- Prompt templates for:
  - Incident type classification (from user's initial message)
  - Follow-up question generation (based on missing fields from procedure)
  - Severity assessment (based on collected info + procedure rules)
  - Incident report summarization

### Step 4.4 — `src/care_companion/features/escalation_agent/nodes.py` (*depends on Steps 4.2, 4.3*)
- LangGraph node functions (async, small, single-purpose):
  - `classify_incident(state)` — detect incident type from user message, load matching procedure
  - `ask_followup(state)` — generate next question based on missing fields
  - `collect_response(state)` — extract answers from user response, update collected_fields
  - `assess_severity(state)` — classify severity based on collected data + procedure rules
  - `generate_notifications(state)` — determine who to notify based on severity
  - `create_report(state)` — compile final `IncidentReport`
  - `confirm_report(state)` — present report to user for confirmation

### Step 4.5 — `src/care_companion/features/escalation_agent/graph.py` (*depends on Step 4.4*)
- Build the LangGraph `StateGraph`:
  - Nodes: classify → ask_followup ↔ collect_response (loop) → assess_severity → generate_notifications → create_report → confirm_report
  - Conditional edges: loop on ask_followup/collect_response until all required fields collected
  - Use LangGraph checkpoints (file-based or SQLite) saved to `data/checkpoints/`
  - Entry point: user's first message
  - Compile graph with checkpoint persistence

### Step 4.6 — `pages/03_Incident_Escalation.py` (*depends on Step 4.5*)
- Streamlit chat interface:
  - Chat-style UI with `st.chat_message`
  - Session state for conversation thread_id (maps to LangGraph checkpoint)
  - "New Incident" button to start fresh
  - "Resume" option that reloads from checkpoint (demonstrates state recovery)
  - Display final incident report in structured format
  - Show escalation status progression (assess → notify → document → resolve)

### Step 4.7 — Write tests for Phase IV
- `tests/features/test_escalation_agent.py`:
  - Full conversation flow: "Patient fell" → follow-up Qs → severity → report
  - State recovery: invoke graph, simulate restart, resume from checkpoint
  - Edge: user provides incomplete answers, agent re-asks
  - Edge: unknown incident type → fallback to general procedure

**Files:** procedure docs in `data/procedures/`, `state.py`, `prompts.py` (escalation), `nodes.py`, `graph.py`, `03_Incident_Escalation.py`
**Depends on:** Phase 0 (models, config); partially on Phase II (Neo4j client for storing reports, but can work independently)
**Verification:** `pytest tests/` passes (especially state recovery test); Streamlit chat works end-to-end for a fall scenario; stop app mid-conversation, restart, resume from checkpoint successfully; final incident report has all required fields per procedure.

---

## Final: Documentation & Polish

### Step 5.1 — `docs/architecture.md`
- System architecture diagram (text-based), component descriptions, data flow

### Step 5.2 — `docs/workflows.md`
- Step-by-step workflow for each feature, including screenshots/descriptions of UI

### Step 5.3 — Final integration verification
- All 3 features work from the Streamlit app
- All tests pass
- Success metrics checked:
  - Normalization rate: verify synonym merging in Neo4j
  - Groundedness: every AI claim has a citation
  - State recovery: escalation agent resumes after restart

---

## Relevant Files (by module)

**Config & Utils**
- `requirements.txt` — all dependencies
- `src/care_companion/config/settings.py` — Pydantic Settings with env vars
- `src/care_companion/utils/env.py` — settings singleton
- `src/care_companion/utils/logging.py` — logger factory

**Data Layer**
- `src/care_companion/data/models.py` — Patient, ProgressNote, Referral, IncidentReport
- `src/care_companion/data/schemas.py` — RiskProfile, ComplianceGap, DischargeReport, EscalationRecord

**Ingestion**
- `scripts/generate_synthetic_data.py` — GPT-powered synthetic data generator
- `src/care_companion/ingestion/note_parser.py` — parse progress notes
- `src/care_companion/ingestion/referral_parser.py` — parse referral PDFs
- `src/care_companion/ingestion/chunking.py` — chunk notes by SOAP section
- `src/care_companion/ingestion/loaders.py` — orchestrator

**Graph (Neo4j)**
- `src/care_companion/graph/neo4j_client.py` — async driver wrapper
- `src/care_companion/graph/normalization.py` — LLM-based entity normalization
- `src/care_companion/graph/triplets.py` — SPO extraction from notes
- `src/care_companion/graph/queries.py` — parameterized Cypher templates

**Retrieval**
- `src/care_companion/retrieval/chroma_store.py` — ChromaDB wrapper
- `src/care_companion/retrieval/hybrid.py` — vector + graph hybrid retrieval
- `src/care_companion/retrieval/citation_builder.py` — source citation formatting

**Features**
- `src/care_companion/features/pre_visit_brief/` — chain.py, prompts.py, service.py
- `src/care_companion/features/discharge_summary/` — chain.py, compliance.py, prompts.py
- `src/care_companion/features/escalation_agent/` — graph.py, nodes.py, prompts.py, state.py

**UI**
- `app.py` — Streamlit entry point
- `pages/01_Pre_Visit_Brief.py`, `pages/02_Discharge_Summary.py`, `pages/03_Incident_Escalation.py`

**Tests**
- `tests/conftest.py` — shared fixtures (mock LLM, mock Neo4j, test patients)
- `tests/features/test_pre_visit_brief.py`, `test_discharge_summary.py`, `test_escalation_agent.py`
- `tests/graph/test_normalization.py`, `tests/retrieval/test_citation_builder.py`

---

## Verification (End-to-End)

1. `pip install -r requirements.txt` — clean install
2. `python scripts/generate_synthetic_data.py` — generates 3–5 patient datasets in `data/`
3. `python scripts/load_sample_data.py` — populates ChromaDB + Neo4j
4. `pytest tests/ -v` — all tests pass
5. `streamlit run app.py` — manual walkthrough:
   - Pre-Visit Brief: select patient → generate → verify risk profile with citations
   - Discharge Summary: select patient → generate → verify compliance gaps are highlighted
   - Incident Escalation: type "Patient fell" → answer follow-ups → verify structured report; restart app → resume conversation
6. Neo4j Browser: verify graph has normalized entities, no duplicates
7. Check that no raw clinical text resembling real PHI is logged

---

## Decisions & Scope

- **Synthetic data format:** Progress notes as JSON (structured SOAP); Referrals as PDF (generated via fpdf2). This matches the PRD's emphasis on PDF handling for referrals.
- **LLM usage:** GPT-4o-mini for normalization and data generation (cost); GPT-4o for final synthesis (quality). Per copilot-instructions.md.
- **Checkpoint storage:** File-based or SQLite in `data/checkpoints/` (simple for learning project, avoids external dependency).
- **Graph layer built in Phase II** even though it's reused in Phase III — the Pre-Visit Brief is the simplest use case to validate the graph pipeline.
- **Excluded:** Real PDF download/export for discharge reports (can be added as stretch goal). Real notification systems for escalation (simulate only).
