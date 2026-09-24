# JD Generation — Multi-Agent System

A Google ADK-based multi-agent application that generates, checks, fixes, and saves
compliant Job Descriptions. Each concern is owned by a dedicated specialist agent,
all coordinated by a central orchestrator.

---

## Agent Architecture

```
root_agent  (Orchestrator / Super-Agent)
  ├── jd_writer_agent      — composes JD text, stores it, returns token
  ├── translation_agent    — translates stored JD into any language (optional)
  ├── compliance_agent     — rule-engine bias / inclusivity / regulatory check
  ├── grader_agent         — quality scoring 0–100 across 5 dimensions
  ├── autofixer_agent      — remediates every issue, updates store
  └── save_agent           — THE ONLY write-to-disk point; returns final reports
```

### Agent responsibilities

| Agent | File | What it does |
|---|---|---|
| **Orchestrator** | `agent.py` | Manages the full pipeline; the only agent the user talks to |
| **JD Writer** | `agents/jd_writer_agent.py` | Composes a structured JD from user input + optional reference docs |
| **Translation** | `agents/translation_agent.py` | Translates the stored JD via Gemini; updates the in-process store |
| **Compliance** | `agents/compliance_agent.py` | Pattern + absence rule engine (bias, inclusivity, regulatory) |
| **Grader** | `agents/grader_agent.py` | Scores the JD on Completeness, Clarity, Inclusivity, Specificity, Compliance |
| **AutoFixer** | `agents/autofixer_agent.py` | Fixes all compliance and quality issues; stores the corrected text |
| **Save** | `agents/save_agent.py` | Runs a final check, writes `.txt` to `outputs/`, cleans up the store |

### Shared modules

| Module | Purpose |
|---|---|
| `jd_store.py` | In-process key-value store; all agents share state via a short token |
| `retry.py` | Exponential-backoff wrapper for Gemini API calls |
| `tools/compliance.py` | Rule definitions + `run_compliance_check()` |
| `tools/grader.py` | Scoring logic + `grade_job_description()` |
| `tools/pdf_reader.py` | Reads PDFs/TXTs from `uploads/` as reference context |

---

## Prerequisites

- Python 3.11+
- A Google API key (Gemini)

---

## Setup

```bash
# 1. Clone / unzip the project
cd jd_multi_agent

# 2. Create and activate a virtual environment
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate

# 3. Install dependencies
pip install -r requirements.txt

# 4. Configure environment
cp .env.example .env
# Edit .env and set GOOGLE_API_KEY=your_key_here
```

---

## Running

### ADK Web UI (recommended)

```bash
# From the PARENT directory of jd_multi_agent/
adk web
```

ADK scans sub-directories for `__init__.py` files that export `root_agent`.
Open http://localhost:8000 and select **jd_orchestrator** from the agent dropdown.

### ADK CLI (single turn)

```bash
adk run jd_multi_agent --message "Generate a JD for a Senior Python Engineer"
```

### Reference documents (optional)

Drop `.pdf` or `.txt` files into `uploads/` before starting the agent.
The JD Writer agent auto-reads them as context (company mission, JD templates, etc.).

---

## Example session

```
User:  Create a JD for a Data Scientist at Acme Corp, London, full-time.

Agent: ✅ Job Description generated successfully!
       Would you like to translate it before saving?
       • Reply YES and the language (e.g. 'Yes, French')
       • Reply NO to save the English version.

User:  No

Agent: ── COMPLIANCE REPORT ────────────────────────
       Status: PASS WITH WARNINGS
       🟡 [REG-S001] Missing salary range ...
       💡 [INCL-D002] Missing EOE statement ...

       ── QUALITY SCORECARD ────────────────────────
       Total: 74 / 100   Grade: B  (Good — address warnings before publishing)
       ...

       ── CHANGES MADE ─────────────────────────────
       💡 Additions:
         • Added equal opportunity employer statement (INCL-D002)
         • Added salary band placeholder (REG-S001)
         • Added GDPR / data-privacy notice (REG-P001)

       ── FINAL COMPLIANCE REPORT ──────────────────
       Status: PASS
       ✅ All checks passed

       ── FINAL QUALITY SCORECARD ──────────────────
       Total: 88 / 100   Grade: A  (Excellent — minor polish optional)
       ...

       ✅ JD saved to: outputs/JD_Data_Scientist_20260610_143022.txt
```

---

## Output files

Saved to `outputs/` with naming:
- English: `JD_<JobTitle>_<YYYYMMDD_HHMMSS>.txt`
- Translated: `JD_<JobTitle>_<Language>_<YYYYMMDD_HHMMSS>.txt`

---

## Extending the system

To add a new capability (e.g. a LinkedIn formatter or ATS keyword checker):

1. Create `agents/my_new_agent.py` with its own `Agent` + tool function(s).
2. Import it in `agents/__init__.py`.
3. Add it to `root_agent`'s `sub_agents` list in `agent.py`.
4. Add a step in the Orchestrator's `ORCHESTRATOR_INSTRUCTION`.

No other files need to change — the shared `jd_store` and `retry` modules are
already available to any new agent.
