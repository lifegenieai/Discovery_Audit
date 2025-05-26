## Final Product Requirements Document

**Project Name:** `discovery-compliance-analyzer`
**Primary User:** Erik Benjaminson – pro se litigant, Snohomish County Superior Court
**Goal:** Generate commissioner-ready exhibits that prove Respondent’s 275-item discovery set violates Washington’s 2024-amended CR 26 (scope, proportionality, certification). The CLI app scores **every** Interrogatory (ROG) and Request for Production (RFP) against a rules-based rubric, using a JSON **case-profile** (claims, data windows, resource gap). Outputs:

1. **`discovery_analysis_summary.md`** – 3 – 4 page narrative declaration.
2. **`discovery_audit_table.md` + `.csv`** – line-by-line audit with metric scores, rationale, rule citations.

---

### 1  Key Assets

| File                    | Purpose                                                                                                                                            |
| ----------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| **`case_profile.json`** | Timeline, children, party resources, **`claims`** (with `asserted_by` flags), and **`data_windows`** (e.g., `"employment_records": "2023-01-01"`). |
| **PDF discovery set**   | Input to be analysed (`--pdf`).                                                                                                                    |
| **`.env`**              | Holds `OPENAI_API_KEY`, `OPENAI_MODEL`, `TOKEN_CAP` (default 2 000 000).                                                                           |

---

### 2  Core User Stories

| # | As a…    | I want…                                                       | So that…                                                                      |
| - | -------- | ------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| 1 | Litigant | `python analyze.py --pdf RFP.pdf --profile case_profile.json` | The tool scores all 275 requests and tells me what it’s doing.                |
| 2 | Litigant | Status lines & progress bar                                   | I know which stage the script is on and how long it may take.                 |
| 3 | Litigant | Clear error messages                                          | I can fix issues (e.g., missing key, rate-limit) without reading a traceback. |
| 4 | Litigant | Three output files                                            | I can attach them to a motion for protective order.                           |
| 5 | Litigant | Token cap enforcement                                         | API cost never surprises me.                                                  |

---

### 3  Functional Requirements

| Stage                         | Required Behaviour                                                                                                                                                                                                                                                            |               |
| ----------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------- |
| **A. Startup**                | Read CLI args, load `.env` and `case_profile.json`. Abort with friendly message if missing.                                                                                                                                                                                   |               |
| **B. PDF Load & OCR**         | Emit `📄 Loading PDF…` then (if needed) `🔍 Running OCR on N pages…`.                                                                                                                                                                                                         |               |
| **C. Extraction**             | Regex-split into numbered items; log counts `✂️ Extracted 275 items`.                                                                                                                                                                                                         |               |
| **D. Domain & Claim Mapping** | • Map each request to domain (employment\_records, banking, parenting, etc.) via keyword+embedding. • Lookup in `claims` – if no match and `contested==false` → flag. • Check `data_windows`: if requested timeframe predates `window_start`, set `WindowViolation = domain`. |               |
| **E. LLM Scoring**            | Batch ≤ 25 items/call to OpenAI. Prompt includes five-metric rubric, baseline facts. Show `🧠 Scoring items X-Y / 275` via `tqdm`. Track live token count; stop if projected > cap.                                                                                           |               |
| **F. Aggregation**            | Compute per-metric averages, pass/fail thresholds.                                                                                                                                                                                                                            |               |
| **G. Report Render**          | Jinja2 templates produce summary MD and audit MD/CSV. Emit `📝 Rendering …`.                                                                                                                                                                                                  |               |
| **H. Completion**             | Print \`✅ Done — Total tokens: …                                                                                                                                                                                                                                              | Runtime: …\`. |
| **I. Error Handling**         | Catch & handle: missing key, PDF I/O, OCR missing, OpenAI errors, token cap, bad JSON schema. Friendly `❌` messages; write full trace to `error.log`, exit with non-zero code.                                                                                                |               |

#### Status Line Palette

| Emoji | Stage         |
| ----- | ------------- |
| 📄    | PDF load      |
| 🔍    | OCR           |
| ✂️    | Extraction    |
| 🧠    | LLM scoring   |
| 📊    | Aggregation   |
| 📝    | Report render |
| ✅     | Success       |
| ❌     | Error / abort |

---

### 4  Non-Functional Requirements

* **Runtime:** ≤ 10 min / 275 items on laptop; memory ≤ 1 GB.
* **Dependencies:** `pdfplumber`, `PyPDF2`, `pytesseract`, `tqdm`, `openai>=1.0`, `pandas`, `jinja2`, `sentence-transformers`, `python-dotenv`, `logging`.
* **Platforms:** Windows 10/11 (VS Code terminal); cross-platform friendly.
* **Security:** Only per-item text sent to OpenAI; PDFs stay local.
* **Logging:** INFO level by default; `--debug` gives DEBUG. All uncaught errors → `error.log`.

---

### 5  Deliverables & Acceptance Criteria

| Deliverable      | Must Pass                                                                                                                   |
| ---------------- | --------------------------------------------------------------------------------------------------------------------------- |
| **`analyze.py`** | Runs end-to-end with sample PDF, respecting token cap, emits status lines.                                                  |
| **Templates**    | Summary MD ≈ 3-4 pages incl. declaration; audit rows = discovery items and have populated `WindowViolation` & `ClaimMatch`. |
| **README.md**    | Install steps, CLI examples, JSON schema.                                                                                   |
| **error.log**    | Captures full trace on any crash.                                                                                           |
| **Unit tests**   | Verify date-window check and claim mapping logic.                                                                           |

---

### 6  High-Level Architecture

```
PDF → (OCR if needed) → Extractor → Domain/Claim Mapper
        ↓                        ↘ uses case_profile.json
        Item list  →  Batched LLM  → Metric scores JSON
                               ↘ token monitor
Scores → pandas DataFrame → Jinja2 → MD/CSV reports
```

---

### 7  Out-of-Scope (v1)

* GUI / web front-end
* Automatic e-filing
* Multi-PDF batch mode
* Redaction or Bates-stamping automation

---

**Success KPI:** User produces court-ready exhibits in one session (< 10 minutes runtime, < \$20 API cost) with clear status feedback and no unhandled crashes.
