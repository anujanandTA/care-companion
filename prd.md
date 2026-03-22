# PRD: Project "Care Companion"

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