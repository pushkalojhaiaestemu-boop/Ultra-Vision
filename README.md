# Autonomous Visual Researcher

> New here? Follow [SETUP.md](SETUP.md) to get it running. Engineering notes (review, architecture, testing, operations) are in [docs/ENGINEERING.md](docs/ENGINEERING.md).

Give the agent an image and a research question. It identifies the subject, works out
what it does not yet know, searches until it does, reconciles sources that disagree,
and writes a report in which every factual sentence traces back to a document.

```
Image
 ↓
Vision Agent            observations, kept separate from identification guesses
 ↓
Entity Identification   a candidate subject, with confidence
 ↓
Research Planner        goals derived from the question
 ↓
Search Agent            queries generated from what is currently missing
 ↓
Source Analyser         provenance classification
 ↓
Extraction Agent        subject-predicate-object claims, each tied to a sentence
 ↓
Claim Verifier          corroborate, detect conflicts, resolve on provenance
 ↓
Knowledge Graph         edges carrying every claim and source behind them
 ↓
Gap Detector            what the question still demands that the graph lacks
 ↓            ↖________ more research needed? ________↲
 ↓
Report Agent            written from the graph, never from the image
 ↓
Grounded Report
```

## Why this is not a multimodal chatbot

A chatbot takes `image → model → paragraph`. The model supplies the facts from its
weights, and the citations, if any, are decoration attached afterwards.

Here the facts live outside the model:

- **The agent decides what to research next.** There is no fixed query sequence. After
  every round the gap detector re-reads the graph and asks what the question still
  demands. The queries you see in the trace are composed at runtime from the gaps.
- **Claims are objects, not prose.** Every claim carries the source, the sentence it
  came from, and a verification status. A claim that loses a conflict is kept on the
  record and excluded from the report, not deleted.
- **The report is generated from the graph.** The report agent never sees the image or
  the raw page text. It receives statements and citation markers. It cannot introduce a
  fact that no source supported, because it is not given one.
- **Disagreement is surfaced.** When two sources give different values for the same
  functional property, the system opens a conflict. It resolves it only when one side
  has decisively stronger provenance. Otherwise the report says so.

## Running it

Nothing here needs an API key.

```bash
# backend
cd backend
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
uvicorn app.main:app --reload --port 8000
```

```bash
# frontend, in a second terminal
cd frontend
npm install
npm run dev          # http://localhost:5173
```

Open the app and pick a prepared subject, or upload your own image and ask anything.

### Without a browser

The whole loop runs in the terminal, which is the quickest way to see the agent think:

```bash
cd backend
python -m app.cli --demo automobile
python -m app.cli --demo architecture
python -m app.cli --demo artwork --ask "Who published the series?"
```

### Tests

```bash
cd backend
python -m unittest discover -s tests -t .   # or: pytest
```

46 tests covering extraction rules, graph provenance, conflict resolution, gap
detection, query escalation, iteration limits, report citation integrity, and
follow-up behaviour.

## Demo mode and real mode

**Demo mode** (the default) is not a replay of a recorded session. It swaps the
providers:

| Provider | Demo mode | Real mode |
|---|---|---|
| Vision | recorded observations for the three prepared plates | multimodal model |
| Search | BM25 retrieval over a small document collection | web search API |
| Extraction | ~30 pattern rules over sentence shape | model, constrained to quote its source sentence |
| Narration | templates composed from evidence items | model, given evidence only |

The retrieval is real ranking, so a different query genuinely returns different
documents, and the agent's choices still matter. The documents themselves are written
for the demo and live on reserved `.example` domains. **No web request is made in demo
mode, and the interface says so.** The prepared images are drawn SVG plates, not
photographs, so nothing copyrighted is bundled in the repository.

Real mode needs three keys in `.env`:

```bash
cp .env.example .env
# set USE_MOCK_MODE=false and fill in VISION_API_KEY, LLM_API_KEY, SEARCH_API_KEY
```

If `USE_MOCK_MODE=false` but keys are missing, the app stays in demo mode rather than
failing at the first search. Search goes through a search API rather than a crawler, so
there is nothing in the codebase that fetches third-party pages directly.

## Where the interesting code is

| Concern | File |
|---|---|
| State machine and the loop | `backend/app/agents/orchestrator.py` |
| What is missing, and why | `backend/app/services/planning/gap_detector.py` |
| Turning a gap into a query | `backend/app/services/planning/query_planner.py` |
| Corroboration and conflict | `backend/app/services/verification/claim_verifier.py` |
| Extraction rules | `backend/app/services/extraction/rules.py` |
| Graph with provenance | `backend/app/graph/repository.py` |
| Grounded report assembly | `backend/app/agents/report_agent.py` |
| Graph-first follow-ups | `backend/app/agents/question_agent.py` |

## Design decisions worth defending

**Functional vs multi-valued predicates.** An early version flagged every repeated
predicate as a conflict, so two museums describing a subject's significance differently
looked like a contradiction. Only functional predicates — those with exactly one true
value, like `production_count` or `completed_in` — can conflict. A subject can be
regarded as several things at once without anyone being wrong.

**Coverage counts distinct predicates, not claims.** Two sources repeating one fact is
still one fact. Counting raw claims let a broad goal like "engineering" close on a
single materials note.

**Breadth-first task scheduling.** Ordering the queue purely by priority let one
stubborn question spend the entire budget while the rest of the plan went untouched.
The queue now covers every open question once before escalating any of them.

**Barren searches end a line of enquiry.** If searching twice returns sources but no new
claims, the task is set aside and the report says so. Spending the remaining budget
re-asking an unanswerable question is worse than admitting the gap.

**Query escalation.** A gap that survives a round gets a more provenance-seeking query:
plain, then `official`, then `official primary source record`, then `archive ledger`.
This is how the Ferrari production conflict gets resolved — the agent goes looking for a
primary source specifically because two secondary ones disagreed.

## Honest limitations

- The demo extraction rules fire on sentence shape. They handle the prepared corpus
  well and will miss plenty of real prose. Real mode hands extraction to a model.
- Source quality scoring is a domain heuristic, not a measure of truth. A government
  register outranking a forum post is a reasonable prior, not evidence.
- Conflict resolution needs a clear provenance margin. Two evenly matched sources leave
  the conflict open by design; see the artwork demo, where a museum record and an
  academic paper disagree on a date and neither wins.
- Diagnostics count process, not accuracy. Twenty-two verified claims means twenty-two
  claims appeared in more than one source, nothing more.
- Sessions live in memory for the life of the process and are archived to SQLite when
  they finish. There is no multi-process story yet.

## Project layout

```
backend/app/
  agents/        orchestrator, vision, research, extraction, graph, report, question
  services/      search, extraction rules, verification, planning, session manager
  providers/     vision / search / llm, each with a mock and a live implementation
  graph/         NetworkX repository — the backend owns all graph state
  storage/       SQLite archive
  api/           FastAPI routes, SSE stream
frontend/src/
  components/    ImageViewer, ResearchPlan, AgentActivity, KnowledgeGraph,
                 SourcePanel, EvidencePanel, EntityExplorer, ReportViewer,
                 ResearchTimeline, ChainRail
  pages/         Landing, Workstation
data/demo/       three simulated document collections and their reference plates
```
