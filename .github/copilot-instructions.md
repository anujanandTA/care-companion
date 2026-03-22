# Project "Care Companion"

## 1. Executive Summary
An 8-week learning project to build three high-impact AI features for home care: Pre-Visit Briefing, Discharge Summarization (Compliance), and Incident Escalation. The goal is to master the transition from simple RAG to multi-document reasoning and stateful agentic workflows.

## 2. Feature Specifications
* **Feature 1: Pre-Visit Brief (RAG)**
    * [cite_start]**Action:** Generate on-demand summaries of progress notes and history.
    * [cite_start]**Output:** Risk profile and time-saving summary for RNs.
* **Feature 2: Discharge Summary (Comparison/Compliance)**
    * [cite_start]**Action:** Summarize care notes AND cross-check them against the initial referral PDF.
    * [cite_start]**Output:** Highlighted compliance gaps and a draft discharge report.
* **Feature 5: Escalation Agent (Stateful Agent)**
    * [cite_start]**Action:** A conversational AI that follows internal procedures to record and escalate incidents.
    * [cite_start]**Output:** Accurate incident records and faster response times.

## 3. Technical Stack (The "Learning" Core)
* **Orchestration:** LangChain / LangGraph.
* **Vector DB:** ChromaDB (Local).
* **LLM:** OpenAI GPT-4o-mini (Cost-effective for testing).
* **UI:** Streamlit.

## 4. 8-Week Roadmap (12 Hours/Week)

### Phase I: The Data Foundation (Weeks 1-2)
* **Tasks:** Generate synthetic notes AND synthetic Referral PDFs.
* **Milestone:** A script that produces a matching pair: a Referral PDF and 5 corresponding Progress Notes.

### Phase II: Feature 1 - Pre-Visit Brief (Weeks 3-4)
* **Tasks:** Build the RAG pipeline to summarize the 5 notes into a "Risk Profile".
* [cite_start]**Milestone:** A dashboard view showing the RN what they need to know before walking into a house.

### Phase III: Feature 2 - Discharge & Compliance (Weeks 5-6)
* **Tasks:** Implement "Multi-Document RAG." The AI reads the Referral and the Notes to see if all requested services were actually delivered.
* [cite_start]**Milestone:** A compliance report highlighting missing actions.

### Phase IV: Feature 5 - Escalation Agent (Weeks 7-8)
* **Tasks:** Use LangGraph to build a decision tree. If a user types "Patient fell," the agent must ask specific follow-up questions required by "procedure."
* **Milestone:** A functional chat interface that records a structured incident report.

## Architecture Principles

### Entity Normalization
- Always normalize clinical synonyms to a single canonical node before inserting into Neo4j (e.g., "SOB" → "Shortness of Breath", "HF" → "Heart Failure").
- Use `gpt-4o-mini` for normalization calls to keep costs low.
- Deduplication must happen at the graph layer — check for existing nodes before creating new ones.

### Knowledge Graph (Neo4j)
- Model data as Subject-Predicate-Object triplets extracted from progress notes.
- Node labels: `Patient`, `Symptom`, `Medication`, `Condition`, `ClinicalEvent`.
- Relationships should be directional and typed (e.g., `[:HAS_SYMPTOM]`, `[:TAKES_MEDICATION]`, `[:TRIGGERS]`).
- Always use parameterized Cypher queries — never interpolate user input into query strings.

### GraphRAG (Hybrid Retrieval)
- Combine vector search (ChromaDB) for semantic context with graph traversal (Neo4j) for factual relationships.
- Every AI-generated claim in a summary **must include a source citation** referencing the specific graph node or note it was derived from.
- Use `gpt-4o` for final synthesis steps.

### LangGraph State Machines
- Use **LangGraph Checkpoints** for all long-running or multi-turn agent workflows, especially the Escalation Agent.
- State must be serializable — no in-memory-only state for anything that needs to survive a restart.
- Model escalation workflows as explicit nodes in the state graph (e.g., `assess → notify → document → resolve`).

## Data & Safety Rules
- **Synthetic data only.** Never reference, load, or simulate real patient data. All patient personas (e.g., "Robert Thompson, 88yo") are AI-generated.
- Do not log or print raw clinical text that could resemble real PHI.
- Validate all inputs at system boundaries (Streamlit forms, API calls).

## Code Conventions
- Python throughout. Use type hints on all function signatures.
- Use `async`/`await` for LangGraph nodes that call LLMs or databases.
- Keep LangGraph node functions small and single-purpose — one responsibility per node.
- Store Neo4j credentials and OpenAI API keys in environment variables; never hardcode them.
- Use `python-dotenv` for local `.env` loading; add `.env` to `.gitignore`.
- Use `pytest` for all testing. Write tests for both expected behavior and edge cases (e.g., missing data, normalization failures).
- Use docstrings in all functions and classes. Include examples where helpful. Docstrings should follow the Numpy style guide.

## Build & Test
```bash
# Install dependencies
pip install -r requirements.txt

# Run Streamlit app
streamlit run app.py

# Run tests
pytest tests/
```

## Success Metrics (reference when validating implementations)
- **Normalization Rate:** % of synonym medical terms correctly merged in the graph.
- **Groundedness Score:** Every AI claim must cite the specific node or note it used.
- **State Recovery:** Escalation agent must resume correctly after a simulated restart.
