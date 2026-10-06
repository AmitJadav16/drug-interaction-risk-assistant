# Drug Interaction Risk Assistant — Implementation Plan

## Top-Level Overview

**Goal:** Build a working MCP-based clinical decision-support prototype that IBM Bob can use as a
load-bearing agentic tool chain. Bob calls our MCP server's individual tools, chains them based on
the patient context, and receives structured JSON findings from each deterministic engine. A final
synthesis tool calls watsonx.ai Granite to produce a ranked, clinician-readable risk report from
those pre-computed structured findings.

**Scope:** A functional vertical slice covering all five analysis dimensions (DDI, comorbidity, PGx,
temporal, clinical evidence) with a synthetic demo patient that exercises the full pipeline
end-to-end. No frontend, no database, no Docker, no auth, no production infrastructure.

**Language:** Python 3.11+
- `ibm-watsonx-ai` SDK is Python-native and mature.
- MCP Python SDK (`mcp`) supports stdio transport directly — no build step.
- Pharmacokinetic math uses only the Python standard library (`math`, `datetime`).
- Judges can run `pip install -r requirements.txt && python server.py` in one shot.

**Architecture summary:**
- IBM Bob invokes our MCP server as a local subprocess over stdio.
- Bob sees seven individually named tools and decides which ones to call and in what order
  (genuine agentic tool use — not a single monolithic dispatcher).
- Each tool runs deterministic logic against embedded JSON knowledge bases and returns structured JSON.
- The `synthesize_risk_report` tool is the only LLM-touching component; it calls watsonx.ai Granite
  with the structured findings as grounded context.
- The LLM never invents half-lives, DDI facts, PGx facts, exposure durations, or clinical evidence.
- Output always indicates whether Granite or the deterministic fallback generated the narrative.

**Repository constraints honoured:**
- All source code lives under `src/`.
- Existing template files outside `src/` (`README.md`, `submission.yaml`, `docs/`, `demo/`,
  `presentation/`, `.gitignore`, `CONTRIBUTING.md`, `.github/`) are preserved and populated —
  never structurally modified or deleted.

---

## Deterministic vs LLM Boundary

| Layer | Mode | Rationale |
|---|---|---|
| Drug name normalization | Deterministic | Regex + brand-to-generic lookup dict |
| DDI lookup | Deterministic | JSON rule table keyed by sorted drug pair |
| Comorbidity check | Deterministic | JSON rule table keyed by condition slug |
| PGx check | Deterministic | JSON rule table; only evaluates user-provided variants |
| Temporal/PK overlap | Deterministic | First-order decay arithmetic on stored half-life data |
| Clinical evidence retrieval | Deterministic | Static JSON citations keyed by drug pair |
| Risk narrative synthesis | LLM — watsonx.ai Granite | Narrates pre-computed structured findings only |

**Hard rule:** The synthesizer prompt embeds all findings as JSON and explicitly instructs the model:
"Do not invent drug interactions, pharmacokinetic values, genetic facts, or clinical evidence
not present in the structured context provided below."

---

## Exposed MCP Tools

| Tool name | Returns | Engine called |
|---|---|---|
| `normalize_drug` | `{canonical: str, original: str, matched_brand: bool}` | normalize.py |
| `check_ddi` | `list[DDIFinding]` | ddi.py |
| `check_comorbidity_risk` | `list[ComorbidityFinding]` | comorbidity.py |
| `check_pgx_risk` | `list[PGxFinding]` | pgx.py |
| `analyze_temporal_overlap` | `list[TemporalFinding]` | temporal.py |
| `get_clinical_evidence` | `list[EvidenceCitation]` | evidence.py |
| `synthesize_risk_report` | `{narrative: str, generated_by: str, findings_summary: dict}` | synthesizer.py |

Each tool is independently callable. Bob decides which to call and how to chain them.
`synthesize_risk_report` accepts the pre-collected findings dict as its input — it never re-runs
the analysis engines internally.

### TemporalFinding Schema

Each `analyze_temporal_overlap` result item contains:

```
{
  proposed_drug:                  str,   // canonical name of the proposed drug
  co_drug:                        str,   // canonical name of the co-administered drug
  proposed_dose_time:             str,   // ISO datetime of the proposed drug dose
  co_drug_dose_time:              str,   // ISO datetime of the co-drug dose being evaluated
  reference_time:                 str,   // the co-drug dose time used as the reference point
  half_life_hours:                float, // PK half-life of the proposed drug (from drug_halflife.json)
  elapsed_hours:                  float, // hours between proposed dose and co-drug dose
  estimated_concentration_fraction: float, // C(T)/C0 = 0.5^(elapsed/half_life); range 0.0–1.0
  overlap_threshold_met:          bool,  // True if estimated_concentration_fraction >= 0.20
  overlap_window:                 str,   // human-readable window description, e.g. "12:00–18:00 (6.0 h)"
  note:                           str    // plain-language explanation of the finding
}
```

**Scope constraint:** All `note` values must be phrased as estimated pharmacokinetic exposure
findings. They must not assert that temporal overlap proves or causes a clinical interaction.

---

## Proposed `src/` Layout

```
src/
├── .env.example                        ← updated with WATSONX_MODEL_ID added
├── README.md                           ← updated to describe actual layout
├── requirements.txt
├── mcp_server/
│   ├── __init__.py
│   ├── server.py                       ← MCP entry point (stdio, FastMCP)
│   ├── tools/
│   │   ├── __init__.py
│   │   ├── normalize.py                ← drug name normalization
│   │   ├── ddi.py                      ← DDI engine
│   │   ├── comorbidity.py              ← comorbidity risk engine
│   │   ├── pgx.py                      ← PGx engine (known variants only)
│   │   ├── temporal.py                 ← temporal/PK overlap engine
│   │   └── evidence.py                 ← clinical evidence retrieval
│   ├── synthesis/
│   │   ├── __init__.py
│   │   └── synthesizer.py              ← watsonx.ai Granite narrator
│   └── data/
│       ├── ddi_rules.json
│       ├── comorbidity_rules.json
│       ├── pgx_rules.json
│       ├── drug_halflife.json
│       └── evidence_citations.json
└── tests/
    ├── __init__.py
    ├── conftest.py                     ← shared test patient fixture
    ├── test_normalize.py
    ├── test_ddi.py
    ├── test_comorbidity.py
    ├── test_pgx.py
    ├── test_temporal.py
    ├── test_evidence.py
    └── test_vertical_slice.py          ← end-to-end integration test
```

---

## Synthetic Data Labeling Policy

All patient data used in demos, tests, and documentation is **synthetic** — fabricated for
illustrative purposes only, not derived from any real patient record.

Every output that includes patient data must include the label:
`"data_provenance": "SYNTHETIC — for demonstration only, not real patient data"`

This label must appear in:
- All test fixtures in `conftest.py`
- The demo patient dict used in `test_vertical_slice.py`
- The `synthesize_risk_report` output `findings_summary` when operating on synthetic data

---

## Clinical Decision-Support Disclaimer

The following disclaimer must appear in every synthesized risk report output (both Granite and
fallback paths), and must be reproduced verbatim in `docs/solution-overview.md` and
`docs/setup-guide.md`:

> **DISCLAIMER:** This tool is a prototype clinical decision-support assistant developed for
> demonstration purposes only. It is not a medical device, not a diagnostic system, and must
> not be used to make prescribing, dispensing, or clinical care decisions. Always consult a
> qualified healthcare professional. Drug interaction data is illustrative and not a substitute
> for a regulatory-approved clinical pharmacology reference.

---

## Minimum Dependencies

```
mcp>=1.0.0              # MCP Python SDK (stdio server, FastMCP)
ibm-watsonx-ai>=1.1.0   # watsonx.ai SDK for Granite inference
python-dotenv>=1.0.0    # .env loading
pytest>=8.0.0           # test runner
```

No database, no web framework, no Docker, no authentication layer.

---

## IBM Bob Integration

1. Bob is configured in its local MCP config to run `python src/mcp_server/server.py` as a subprocess.
2. Communication is via **stdio MCP protocol** — Bob calls named tools with JSON arguments.
3. Bob's system prompt describes the patient scenario; Bob decides which tools to call and in what sequence.
4. A representative Bob session for the demo scenario:
   - Bob calls `normalize_drug` for each medication name in the patient's list.
   - Bob calls `check_ddi` with the canonical current drugs + proposed drug.
   - Bob calls `check_comorbidity_risk` with conditions + proposed drug.
   - Bob calls `check_pgx_risk` with known variants + all drugs.
   - Bob calls `analyze_temporal_overlap` with the medication schedule.
   - Bob calls `get_clinical_evidence` for the highest-severity DDI pair.
   - Bob calls `synthesize_risk_report` with all collected findings.
5. The IBM Bob MCP config entry (for local use):
   ```json
   {
     "drug-interaction-assistant": {
       "command": "python",
       "args": ["src/mcp_server/server.py"],
       "cwd": "<repo root>"
     }
   }
   ```

---

## Synthetic Demo Test Case

**Patient profile (synthetic — not a real patient):**
- Age: 72
- Conditions: Type 2 Diabetes, Atrial Fibrillation
- Genetic variants: CYP2C9*2 (provided by user)
- Current medications:
  - Metformin 500 mg, twice daily — dosed at 08:00 and 20:00
  - Warfarin 5 mg, once daily — dosed at 18:00
- Proposed new medication: Fluconazole 150 mg, once daily — proposed dose time 12:00

**Expected findings:**

| Engine | Finding |
|---|---|
| DDI | HIGH: Fluconazole is a strong CYP2C9 inhibitor → increases Warfarin plasma concentration → elevated bleeding risk |
| Comorbidity | WARNING: Atrial Fibrillation patient on Warfarin — adding a CYP2C9 inhibitor significantly raises anticoagulation risk |
| PGx | HIGH: CYP2C9*2 variant already reduces Warfarin metabolism; Fluconazole compounds this further |
| Temporal | OVERLAP: Fluconazole dosed at 12:00 (t½ ≈ 30h) is at ~87% peak concentration at Warfarin dose time 18:00 (6h later); active overlap confirmed |
| Evidence | FDA Diflucan label; pharmacokinetic study showing ~50% increase in Warfarin AUC |

**Temporal calculation verification:**
`C(6h) = 1.0 × 0.5^(6/30) = 0.5^0.2 ≈ 0.871` → 87.1% of peak concentration remaining at Warfarin dose time.

---

## Sub-Task 1 — Project Scaffold and Dependencies

**Status:** [ ] pending

### Intent
Create the foundational file structure under `src/` and the dependency manifest. Nothing else
runs until this exists.

### Expected Outcomes
- `src/requirements.txt` exists with the four required packages.
- `src/mcp_server/` package skeleton exists (all `__init__.py` files, empty module stubs).
- `src/mcp_server/data/` directory exists with empty stub JSON files (each `{}`).
- `src/tests/` directory exists.
- `src/.env.example` updated: `WATSONX_MODEL_ID=ibm/granite-3-3-8b-instruct` added below the
  existing watsonx block; all existing lines kept intact.
- `src/README.md` updated to describe the actual `src/` folder tree.

### Todo List
- [ ] Create `src/requirements.txt` with: `mcp>=1.0.0`, `ibm-watsonx-ai>=1.1.0`, `python-dotenv>=1.0.0`, `pytest>=8.0.0`
- [ ] Create `src/mcp_server/__init__.py` (empty)
- [ ] Create `src/mcp_server/tools/__init__.py` (empty)
- [ ] Create `src/mcp_server/synthesis/__init__.py` (empty)
- [ ] Create stub files: `src/mcp_server/tools/normalize.py`, `ddi.py`, `comorbidity.py`, `pgx.py`, `temporal.py`, `evidence.py` — each with a single `# TODO` comment
- [ ] Create `src/mcp_server/synthesis/synthesizer.py` — stub with `# TODO`
- [ ] Create `src/mcp_server/data/` with empty stub JSON files: `ddi_rules.json`, `comorbidity_rules.json`, `pgx_rules.json`, `drug_halflife.json`, `evidence_citations.json` — each as `{}`
- [ ] Create `src/tests/__init__.py` (empty)
- [ ] Update `src/.env.example`: add `WATSONX_MODEL_ID=ibm/granite-3-3-8b-instruct` below the existing watsonx block
- [ ] Update `src/README.md` with actual folder tree and brief description of each module

### Relevant Context
- `src/.env.example` already has `WATSONX_API_KEY`, `WATSONX_PROJECT_ID`, `WATSONX_URL` — keep all.
- The MCP Python SDK package is named `mcp` on PyPI.
- Do not create `src/mcp_server/server.py` yet — that is Sub-Task 2.

---

## Sub-Task 2 — MCP Server Foundation

**Status:** [ ] pending

### Intent
Implement `src/mcp_server/server.py` — the MCP server that IBM Bob connects to over stdio.
This sub-task creates the server skeleton with all seven tools registered (stubs only) so that
IBM Bob can connect and discover the tool list immediately, before the engines are implemented.

### Expected Outcomes
- `python src/mcp_server/server.py` starts a valid MCP server on stdio with no errors.
- All seven tools are registered using `@mcp.tool()` with typed parameters and clear docstrings.
- Each tool handler returns a placeholder `{"status": "not_implemented"}` response until its
  engine is implemented in later sub-tasks.
- `src/.bob/mcp.json` is created at the repository root level (outside `src/`) with the correct
  config for Bob to connect to this server.
- A `src/mcp_server/DISCLAIMER.py` constant module is created holding the verbatim disclaimer
  string, importable by all tools and the synthesizer.

### MCP Configuration File
Create `.bob/mcp.json` at the **repository root** (not inside `src/`) with this content:
```json
{
  "mcpServers": {
    "drug-interaction-assistant": {
      "command": "python",
      "args": ["src/mcp_server/server.py"],
      "env": {}
    }
  }
}
```
The `cwd` defaults to wherever Bob is run from — judges must run Bob from the repository root.

### Seven Tool Registrations
```
normalize_drug            → normalize_drug_name()
check_ddi                 → check_ddi()
check_comorbidity_risk    → check_comorbidity()
check_pgx_risk            → check_pgx()
analyze_temporal_overlap  → analyze_temporal_overlap()
get_clinical_evidence     → get_clinical_evidence()
synthesize_risk_report    → synthesize_risk_report()
```

### Todo List
- [ ] Create `src/mcp_server/DISCLAIMER.py` with `DISCLAIMER_TEXT` constant (verbatim text from plan)
- [ ] Create `src/mcp_server/server.py` using `from mcp.server.fastmcp import FastMCP`
- [ ] Instantiate: `mcp = FastMCP("drug-interaction-assistant")`
- [ ] Register all seven tools as stubs: typed parameters, clear docstrings, placeholder return value
- [ ] Add `if __name__ == "__main__": mcp.run()` entry point
- [ ] Create `.bob/mcp.json` at repository root with the MCP server configuration shown above
- [ ] Manually verify: `python src/mcp_server/server.py` starts without import errors

### Relevant Context
- FastMCP tool docstrings become the tool descriptions Bob sees — write them clearly and specifically.
  Bad: "Analyzes drugs." Good: "Given a list of current canonical drug names and one proposed drug name,
  returns all drug-drug interaction findings from the embedded DDI knowledge base."
- The server runs over stdio — no uvicorn, no HTTP port, no FastAPI.
- Stub tool handlers must import from `tools/` and `synthesis/` modules but can guard the call
  with a `try/except NotImplementedError` until each module is filled in.
- `.bob/mcp.json` is the file IBM Bob reads to discover and launch MCP servers.

---

## Sub-Task 3 — Embedded Clinical Knowledge Bases

**Status:** [ ] pending

### Intent
Populate the five JSON data files. These are static, embedded reference tables — no external API
calls. They must cover the demo test case completely plus a small number of additional entries for
breadth. All PK values must cite their source in a `_source` field.

### Expected Outcomes
- `ddi_rules.json`: Fluconazole↔Warfarin interaction (HIGH, CYP2C9 inhibition) plus at minimum 3
  additional interaction pairs for breadth.
- `comorbidity_rules.json`: Atrial Fibrillation + CYP2C9 inhibitor + anticoagulant warning, plus
  2–3 other condition entries.
- `pgx_rules.json`: CYP2C9*2 and CYP2C9*3 entries affecting Warfarin and Fluconazole, plus
  CYP2C19*2 affecting clopidogrel for breadth.
- `drug_halflife.json`: Warfarin (~40h), Fluconazole (~30h), Metformin (~6h), plus 3–4 others.
- `evidence_citations.json`: FDA Diflucan label + PK study for Fluconazole-Warfarin pair.

### Data Schema Conventions
```
ddi_rules.json:          keyed by "drug_a|drug_b" (alphabetically sorted, lowercase)
comorbidity_rules.json:  keyed by condition-slug (lowercase, hyphenated)
pgx_rules.json:          keyed by "GENE*ALLELE" (uppercase gene, e.g. "CYP2C9*2")
drug_halflife.json:      keyed by canonical drug name (lowercase)
evidence_citations.json: keyed by "drug_a|drug_b" (same convention as ddi_rules)
```

### Todo List
- [ ] Write `src/mcp_server/data/ddi_rules.json` — severity levels: CRITICAL / HIGH / MODERATE / LOW
- [ ] Write `src/mcp_server/data/comorbidity_rules.json`
- [ ] Write `src/mcp_server/data/pgx_rules.json`
- [ ] Write `src/mcp_server/data/drug_halflife.json` — each entry: `{half_life_hours, typical_dose_mg, typical_frequency_per_day, _source}`
- [ ] Write `src/mcp_server/data/evidence_citations.json` — each citation: `{source, year, summary, url_or_ref, evidence_level}`

### Relevant Context
- Evidence level taxonomy: `A` (RCT/meta-analysis), `B` (controlled studies), `C` (case series),
  `D` (FDA label / regulatory document).
- Drug pair keys must always be alphabetically sorted so lookup is order-independent.
- PK values must reflect established clinical references (FDA labels, pharmacology references).
  The `_source` field is mandatory for every entry in `drug_halflife.json`.
- Keep scope minimal: demo drugs + ~4 extras. Do not inflate the knowledge base beyond what is needed.

---

## Sub-Task 4 — Drug Normalization Tool

**Status:** [ ] pending

### Intent
Implement `src/mcp_server/tools/normalize.py`. This is the gateway module — all other engines
call it to get a canonical lowercase drug name from free-text input.

### Expected Outcomes
- `normalize_drug_name(raw: str) -> dict` returns `{canonical, original, matched_brand}`.
- Handles brand names: Coumadin → warfarin, Diflucan → fluconazole, Glucophage → metformin.
- Strips dose and units ("Warfarin 5mg" → "warfarin").
- Returns the cleaned lowercased string if no brand mapping found (graceful fallback).
- Unit tests pass.

### MCP Tool Interface
```
Tool: normalize_drug
Input:  {drug_name: str}
Output: {canonical: str, original: str, matched_brand: bool}
```

### Todo List
- [ ] Implement `normalize_drug_name(raw: str) -> dict` in `src/mcp_server/tools/normalize.py`
- [ ] Define `BRAND_TO_GENERIC` dict covering demo drugs + ~10 common brand names
- [ ] Dose-stripping regex: remove trailing `\s+\d+(\.\d+)?\s*(mg|mcg|g|ml|units?|iu)` (case-insensitive)
- [ ] Write `src/tests/test_normalize.py` covering: brand name, dose stripping, unknown drug fallback, case insensitivity

### Relevant Context
- This module must not import `ibm-watsonx-ai` — purely deterministic.
- The returned `canonical` value is what all other engines use as their key.
- Replace the stub in `server.py` `normalize_drug` handler with the real call after this sub-task is complete.

---

## Sub-Task 5 — DDI Engine

**Status:** [ ] pending

### Intent
Implement `src/mcp_server/tools/ddi.py`. Given canonical current drug names and a canonical
proposed drug name, look up each pair in `ddi_rules.json` and return structured findings.

### Expected Outcomes
- `check_ddi(current_drugs: list[str], proposed_drug: str) -> list[dict]` exists.
- Each finding: `{drug_a, drug_b, severity, mechanism, effect, recommended_action}`.
- Returns empty list when no interactions found — not an error.
- Correctly detects fluconazole-warfarin HIGH interaction.
- Results sorted by severity (CRITICAL → HIGH → MODERATE → LOW).
- Unit tests pass.

### MCP Tool Interface
```
Tool: check_ddi
Input:  {current_drugs: list[str], proposed_drug: str}
Output: list of {drug_a, drug_b, severity, mechanism, effect, recommended_action}
```

### Todo List
- [ ] Implement `check_ddi` in `src/mcp_server/tools/ddi.py`
- [ ] Load `ddi_rules.json` once at module import (not on every call)
- [ ] For each (current_drug, proposed_drug) pair: normalize both names, sort alphabetically, look up in rules
- [ ] Sort returned findings by severity
- [ ] Write `src/tests/test_ddi.py`: interaction found, no interaction, unknown drug (graceful skip), sorting order

### Relevant Context
- All inputs must be passed through `normalize_drug_name` before lookup.
- `check_ddi` accepts already-normalized names; normalization is the caller's responsibility at the
  MCP tool layer (server.py calls normalize before calling the engine functions).

---

## Sub-Task 6 — Comorbidity Risk Engine

**Status:** [ ] pending

### Intent
Implement `src/mcp_server/tools/comorbidity.py`. Given patient conditions and the proposed drug
(plus current drugs for context), identify condition-drug risk combinations.

### Expected Outcomes
- `check_comorbidity(conditions: list[str], proposed_drug: str, current_drugs: list[str]) -> list[dict]`
- Each finding: `{condition, drug, risk_level, reason, recommended_action}`.
- Correctly flags AF + CYP2C9 inhibitor + Warfarin scenario.
- Condition names normalised to hyphen-slug (e.g. "Atrial Fibrillation" → "atrial-fibrillation").
- Unit tests pass.

### MCP Tool Interface
```
Tool: check_comorbidity_risk
Input:  {conditions: list[str], proposed_drug: str, current_drugs: list[str]}
Output: list of {condition, drug, risk_level, reason, recommended_action}
```

### Todo List
- [ ] Implement `check_comorbidity` in `src/mcp_server/tools/comorbidity.py`
- [ ] Implement `normalize_condition(name: str) -> str` — lowercase + hyphen-slug + alias dict (e.g. "AF" → "atrial-fibrillation", "T2DM" → "type-2-diabetes")
- [ ] Load `comorbidity_rules.json` at module import
- [ ] Write `src/tests/test_comorbidity.py`

### Relevant Context
- The comorbidity rules may flag a combination of condition + proposed_drug, or a combination of
  condition + proposed_drug + a specific current_drug (three-way interaction). The JSON schema
  must support both patterns.

---

## Sub-Task 7 — PGx Engine

**Status:** [ ] pending

### Intent
Implement `src/mcp_server/tools/pgx.py`. Given explicitly user-provided genetic variants, assess
pharmacogenomic risk for the drugs in context. Only evaluate variants that the user has provided —
do not recommend which genetic tests to order or scan for potential unknown variants.

### Expected Outcomes
- `check_pgx(variants: list[str], proposed_drug: str, current_drugs: list[str]) -> list[dict]`
- Each finding: `{gene, variant, drug, impact_level, mechanism, recommendation}`.
- If `variants` is empty: returns `[{note: "No genetic variants provided. PGx analysis skipped."}]` — clearly signals "not run" rather than "no risk".
- If a variant is provided but not in the knowledge base: returns a per-variant note indicating it is not in the supported variant set.
- Correctly handles CYP2C9*2 → Warfarin (and CYP2C9 inhibitor Fluconazole compounding it).
- Unit tests pass.

### MCP Tool Interface
```
Tool: check_pgx_risk
Input:  {variants: list[str], proposed_drug: str, current_drugs: list[str]}
Output: list of {gene, variant, drug, impact_level, mechanism, recommendation}
        OR [{note: str}] for the no-variants / unsupported-variant cases
```

### Variant Format
- Standard format: `GENE*ALLELE` (e.g. `CYP2C9*2`, `CYP2C19*2`).
- Normalise to uppercase gene name before lookup.
- Accept whitespace variants like `CYP2C9 *2` → `CYP2C9*2`.

### Todo List
- [ ] Implement `check_pgx` in `src/mcp_server/tools/pgx.py`
- [ ] Implement `normalize_variant(v: str) -> str` — strip spaces around `*`, uppercase
- [ ] Load `pgx_rules.json` at module import
- [ ] Handle: empty variants list, unsupported variant (not in JSON), matching variant with drug hit
- [ ] Both `proposed_drug` and all `current_drugs` are checked against each variant's impact map
- [ ] Write `src/tests/test_pgx.py`

### Relevant Context
- The "no variants provided" path returns a clearly-labelled note, not an empty list, so that the
  synthesizer can distinguish "no risk found" from "analysis not run".
- PGx findings apply to both the proposed drug AND current drugs (a variant may already affect an
  existing medication; adding a new drug that shares or inhibits the same pathway compounds the risk).

---

## Sub-Task 8 — Temporal Overlap Engine

**Status:** [ ] pending

### Intent
Implement `src/mcp_server/tools/temporal.py` — the core differentiator of this project. Given
medication dose times and PK half-life reference data, compute estimated plasma concentration
overlap between the proposed drug and each current drug at their respective dosing windows.

**Important scope constraint:** The temporal engine reports estimated pharmacokinetic exposure
overlap based on stored reference data and provided administration times. It does NOT claim that
temporal overlap proves a clinical interaction — that inference is left to the synthesizer.

### Expected Outcomes
- `analyze_temporal_overlap(medication_schedule: list[dict], proposed_entry: dict) -> list[dict]`
- `medication_schedule` item: `{drug: str, dose_time_iso: str, dose_mg: float, frequency_per_day: int}`
- `proposed_entry`: same format.
- Each finding contains all fields from the **TemporalFinding Schema** defined in the Exposed MCP
  Tools section: `proposed_drug`, `co_drug`, `proposed_dose_time`, `co_drug_dose_time`,
  `reference_time`, `half_life_hours`, `elapsed_hours`, `estimated_concentration_fraction`,
  `overlap_threshold_met`, `overlap_window`, `note`.
- Overlap threshold: `estimated_concentration_fraction` ≥ 0.20 → `overlap_threshold_met: True`.
- Correctly computes: Fluconazole 12:00 (t½=30h) at Warfarin 18:00 → 0.5^(6/30) ≈ 0.871.
- `overlap_window` populated as a human-readable string, e.g. `"12:00–18:00 (6.0 h)"`.
- All PK values read from `drug_halflife.json` — no values hardcoded in function logic.
- If a drug is not in `drug_halflife.json`, returns a per-drug note: "PK data not available for X".
- Unit tests pass.

### Calculation Model
```
C(T) = C0 × 0.5^(elapsed_hours / half_life_hours)   [first-order decay]
```

Analyse over one full 24-hour dosing cycle (all dose windows of each drug pair).

### MCP Tool Interface
```
Tool: analyze_temporal_overlap
Input:  {
  medication_schedule: list[{drug, dose_time_iso, dose_mg, frequency_per_day}],
  proposed_entry: {drug, dose_time_iso, dose_mg, frequency_per_day}
}
Output: list of TemporalFinding dicts
```

### Todo List
- [ ] Implement `analyze_temporal_overlap` in `src/mcp_server/tools/temporal.py`
- [ ] Implement `concentration_at_time(half_life_hours: float, elapsed_hours: float) -> float` as a pure function
- [ ] Parse ISO dose times with Python `datetime` (no extra date library)
- [ ] For repeating doses: generate all dose windows across a 24h horizon; find the minimum elapsed time between proposed dose and each co-drug dose window
- [ ] Load `drug_halflife.json` at module import; handle missing drug gracefully
- [ ] Write `src/tests/test_temporal.py`: overlapping case, non-overlapping (short half-life), same dose time, missing PK data

### Relevant Context
- "dose_time_iso" may be a full ISO datetime or a time-of-day string like "12:00". If only
  time-of-day is given, use a fixed reference date (e.g. `2024-01-01`) for arithmetic.
- For repeating doses (frequency_per_day > 1), generate all dose windows: e.g. 08:00 and 20:00
  for frequency_per_day=2 starting from the given dose_time_iso.

---

## Sub-Task 9 — Clinical Evidence Retrieval Tool

**Status:** [ ] pending

### Intent
Implement `src/mcp_server/tools/evidence.py`. Retrieves structured clinical evidence citations
from `evidence_citations.json` for a given drug pair.

### Expected Outcomes
- `get_clinical_evidence(drug_a: str, drug_b: str) -> list[dict]` exists.
- Returns list of `{source, year, summary, url_or_ref, evidence_level}`.
- Returns empty list when no citations exist (not an error).
- Correctly returns citations for fluconazole-warfarin.
- Unit tests pass.

### MCP Tool Interface
```
Tool: get_clinical_evidence
Input:  {drug_a: str, drug_b: str}
Output: list of {source, year, summary, url_or_ref, evidence_level}
```

### Todo List
- [ ] Implement `get_clinical_evidence` in `src/mcp_server/tools/evidence.py`
- [ ] Normalise and sort drug pair key (same alphabetic convention as DDI engine)
- [ ] Load `evidence_citations.json` at module import
- [ ] Write `src/tests/test_evidence.py`

### Relevant Context
- Evidence level taxonomy: `A`, `B`, `C`, `D` (see Sub-Task 2 for definitions).
- The drug names passed in may be raw text — normalize before key construction.

---

## Sub-Task 10 — Risk Synthesizer (watsonx.ai)

**Status:** [ ] pending

### Intent
Implement `src/mcp_server/synthesis/synthesizer.py`. The only LLM-touching module. Takes the
pre-computed structured findings from all engines and calls watsonx.ai Granite to produce a
ranked, clinician-readable narrative. The output always declares which generator produced it.

### Expected Outcomes
- `synthesize_risk_report(findings: dict) -> dict` returns `{narrative: str, generated_by: str, findings_summary: dict}`
- `generated_by` is one of: `"watsonx-granite"` or `"deterministic-fallback"` — never omitted.
- `findings` dict keys: `patient_context`, `ddi_findings`, `comorbidity_findings`, `pgx_findings`, `temporal_findings`, `evidence_citations`.
- The prompt explicitly instructs the model: "Do not invent drug interactions, pharmacokinetic values, genetic facts, or clinical evidence not present in the structured context below."
- The prompt instructs the model to include the disclaimer: "This output is from a prototype clinical decision-support tool and must not be used for clinical decision-making."
- Fallback path (no WATSONX_API_KEY): formats the structured findings into a plain-text summary and sets `generated_by = "deterministic-fallback"`.
- Fallback never silently masquerades as LLM output.
- Unit tests pass (fallback path tested without credentials; live Granite path marked `@pytest.mark.integration`).

### MCP Tool Interface
```
Tool: synthesize_risk_report
Input:  {
  patient_context: dict,
  ddi_findings: list,
  comorbidity_findings: list,
  pgx_findings: list,
  temporal_findings: list,
  evidence_citations: list
}
Output: {narrative: str, generated_by: str, findings_summary: dict}
```

### Todo List
- [ ] Implement `synthesize_risk_report` in `src/mcp_server/synthesis/synthesizer.py`
- [ ] Load watsonx credentials from environment using `python-dotenv`
- [ ] Build a structured prompt template embedding all findings as JSON
- [ ] Call `ibm_watsonx_ai.ModelInference` (or equivalent current SDK class) with `model_id` from `WATSONX_MODEL_ID` env var, defaulting to `ibm/granite-3-3-8b-instruct`
- [ ] Implement fallback path: format findings into plain-text, set `generated_by = "deterministic-fallback"`
- [ ] Write `src/tests/test_synthesizer.py`: test fallback path; mark live test `@pytest.mark.integration`

### Relevant Context
- Model: `ibm/granite-3-3-8b-instruct` (via `WATSONX_MODEL_ID`).
- SDK pattern (current): `from ibm_watsonx_ai import Credentials` then `ModelInference(model_id=..., credentials=..., project_id=...)`.
- The synthesizer must never call any of the engine functions internally — it only narrates the findings dict passed to it.
- The `findings_summary` in the output is a pass-through of the input findings (structured, for Bob to parse if needed).

---

## Sub-Task 11 — Wire Up MCP Server with Real Engines

**Status:** [ ] pending

### Intent
Replace all stub handlers in `src/mcp_server/server.py` with the real engine function calls now
that all engines are implemented. This sub-task also adds a live Bob/MCP end-to-end test that
confirms Bob can actually connect to, discover, and invoke the server tools.

### Expected Outcomes
- All seven `@mcp.tool()` handlers in `server.py` call their real engine functions.
- Each handler normalizes its inputs (drug names, variants, conditions) before passing to engines.
- Each handler appends the `DISCLAIMER_TEXT` constant to the tool output where appropriate.
- `python src/mcp_server/server.py` starts without import errors.
- `src/tests/test_mcp_server.py` confirms: server starts, tools list is discoverable, `normalize_drug` returns expected result for "Coumadin 5mg", `check_ddi` returns a HIGH finding for fluconazole + warfarin.

### MCP End-to-End Test Approach
The test launches the server as a subprocess (using `mcp` SDK's test client or `subprocess` +
stdin/stdout), sends an MCP `tools/list` request, and sends a `tools/call` request for
`normalize_drug`. This validates the full MCP wire protocol path without requiring a live Bob session.

### Todo List
- [ ] Replace all stub handlers in `server.py` with real engine calls
- [ ] Each handler: normalize inputs → call engine → return JSON-serializable result
- [ ] `synthesize_risk_report` handler: import `DISCLAIMER_TEXT`; append disclaimer to `narrative` field
- [ ] Create `src/tests/test_mcp_server.py` with MCP subprocess/client test for tools discovery and at least two tool calls
- [ ] Verify: `python src/mcp_server/server.py` starts without errors
- [ ] Verify: `pytest src/tests/test_mcp_server.py -v` passes

### Relevant Context
- The MCP Python SDK provides `mcp.client.stdio` utilities for spawning and talking to a stdio server in tests.
- The server must handle all inputs as strings (MCP JSON transport); type coercion from string to float/int happens inside the handler before passing to engine functions.
- Do not add an HTTP layer — keep stdio only.

---

## Sub-Task 12 — Synthetic Test Case and Integration Test

**Status:** [ ] pending

### Intent
Write `src/tests/test_vertical_slice.py` — the end-to-end integration test using the demo patient
scenario. Also write `src/tests/conftest.py` with the shared patient fixture.

### Test Case (Synthetic Patient — Not a Real Patient)
```python
patient = {
    "age": 72,
    "conditions": ["type-2-diabetes", "atrial-fibrillation"],
    "genetic_variants": ["CYP2C9*2"],
    "current_medications": [
        {"drug": "metformin", "dose_mg": 500, "frequency_per_day": 2, "dose_time_iso": "2024-01-01T08:00:00"},
        {"drug": "warfarin",  "dose_mg": 5,   "frequency_per_day": 1, "dose_time_iso": "2024-01-01T18:00:00"},
    ]
}
proposed = {"drug": "fluconazole", "dose_mg": 150, "frequency_per_day": 1, "dose_time_iso": "2024-01-01T12:00:00"}
```

### Required Assertions
- DDI: at least one finding with severity HIGH or CRITICAL for fluconazole-warfarin pair.
- Comorbidity: at least one finding referencing "atrial-fibrillation".
- PGx: at least one finding for CYP2C9*2 affecting warfarin.
- Temporal: fluconazole-warfarin finding has `overlap_threshold_met = True` and `estimated_concentration_fraction >= 0.80`.
- Temporal: fluconazole-warfarin finding contains all required TemporalFinding fields including `overlap_window`, `proposed_dose_time`, `co_drug_dose_time`, `reference_time`, `elapsed_hours`, `half_life_hours`.
- Evidence: citations list for fluconazole-warfarin contains ≥ 1 entry.
- Synthesizer fallback: completes without exception, `generated_by == "deterministic-fallback"`, narrative contains the verbatim disclaimer text.
- Synthesizer fallback: `findings_summary` contains `data_provenance == "SYNTHETIC — for demonstration only, not real patient data"`.

### Todo List
- [ ] Create `src/tests/conftest.py` with `demo_patient` and `demo_proposed` pytest fixtures; include `"data_provenance": "SYNTHETIC — for demonstration only, not real patient data"` in both fixtures
- [ ] Create `src/tests/test_vertical_slice.py` testing each engine against the demo scenario
- [ ] Add assertion that all temporal findings contain the full TemporalFinding schema fields
- [ ] Mark the live watsonx synthesizer test `@pytest.mark.integration`
- [ ] Verify: `cd src && pip install -r requirements.txt && pytest tests/ -v -m "not integration"` passes

### Relevant Context
- The synthesizer fallback test must run with `WATSONX_API_KEY` unset — do not skip it.
- All engine tests in the vertical slice operate directly on engine functions, not through the MCP server (no subprocess needed for unit/integration tests).
- `test_mcp_server.py` (Sub-Task 11) covers the MCP wire protocol; `test_vertical_slice.py` covers engine correctness.

---

## Sub-Task 13 — Submission Metadata and Documentation

**Status:** [ ] pending

### Intent
Fill in the required hackathon submission files so the GitHub Actions validator passes. Team
metadata (exact name/email) will be finalised separately — placeholders for those specific fields
are acceptable in this sub-task.

### Expected Outcomes
- `submission.yaml`: all required fields filled except `team.lead.name` and `team.lead.email`
  (those remain as `[MY NAME]` / `[MY EMAIL]` placeholders until provided); `track: AI`;
  `ibm_technologies` includes IBM Bob and watsonx.ai.
- `README.md`: all `[placeholder]` text replaced; How to Run section matches setup guide.
- `docs/problem-statement.md`: written.
- `docs/solution-overview.md`: written.
- `docs/architecture.md`: updated with actual Mermaid diagram.
- `docs/setup-guide.md`: exact, tested install and run commands.
- `demo/live-demo-url.txt`: set to `NOT DEPLOYED`.

### Todo List
- [ ] Fill `submission.yaml` (track: AI, title, problem_statement, solution_summary, key_features, tech stack)
- [ ] Replace all `[placeholder]` in `README.md`
- [ ] Write `docs/problem-statement.md`
- [ ] Write `docs/solution-overview.md`
- [ ] Write `docs/architecture.md` with actual Mermaid architecture diagram
- [ ] Write `docs/setup-guide.md` with exact install and run commands
- [ ] Update `demo/live-demo-url.txt` to `NOT DEPLOYED`

### Relevant Context
- GitHub Actions validator will still flag `demo/demo-video-link.txt` until a real video URL is
  added — this is expected and will be resolved when the demo is recorded.
- `submission.yaml` must not have `[MY NAME]` or `[MY EMAIL]` at final submission — these are
  temporary placeholders only.
- The setup guide must include the Bob MCP config snippet (the `.bob/mcp.json` content) so judges
  can connect Bob to the server.
- The verbatim disclaimer must appear in `docs/solution-overview.md` and `docs/setup-guide.md`.

---

## Implementation Order

```
1 (scaffold/deps) → 2 (MCP foundation + .bob/mcp.json) → 3 (knowledge bases) → 4 (normalize) → 5 (DDI) → 6 (comorbidity) → 7 (PGx) → 8 (temporal) → 9 (evidence) → 10 (Granite synthesis) → 11 (wire up + MCP e2e test) → 12 (vertical slice integration tests) → 13 (docs)
```

Sub-tasks 5 through 9 (the five engines) are mutually independent after Sub-tasks 1–3 are done
and can be implemented in any order.

---

## Key Design Decisions (Confirmed)

| Decision | Choice | Rationale |
|---|---|---|
| Language | Python 3.11 | ibm-watsonx-ai SDK is Python-native; no build step; minimal judge setup |
| MCP transport | stdio | Required for local IBM Bob integration; no HTTP port needed |
| Tool granularity | 7 individual tools | Enables genuine agentic chaining by Bob; no monolithic dispatcher |
| MCP config | `.bob/mcp.json` at repo root | Bob discovers and launches the server from this file |
| Knowledge bases | Embedded JSON files | No external API, no database, deterministic and fully reproducible |
| LLM boundary | synthesizer.py only | All PK values and clinical facts are deterministic; LLM only narrates |
| PGx scope | Known variants only | No blind scan; unsupported variants explicitly flagged as unsupported |
| Temporal scope | Exposure overlap only | Reports PK overlap; does not assert clinical interaction causation |
| Temporal output | Full TemporalFinding schema | All timing and PK fields explicit; no hidden calculations |
| Synthesizer fallback | Plain-text, labelled | Works without watsonx credentials; never silently masquerades as LLM |
| Generated-by field | Always present | Every synthesis output declares its generator transparently |
| Synthetic data label | data_provenance field | Every demo/test output carries explicit synthetic-data marker |
| Clinical disclaimer | Verbatim in all outputs | Mandatory in synthesizer output and in docs |
| PK model | First-order decay | Standard pharmacokinetic simplification; appropriate for prototype scope |
| Testing | pytest unit + integration + MCP e2e | Unit tests per engine; MCP wire protocol test; vertical slice integration test |

---

## What This System Is Not

- Not a medical device or diagnostic system.
- Not a replacement for a clinical pharmacist, physician, or regulatory-approved DDI database.
- Not a production system — prototype scope only.
- All tool outputs include or support the disclaimer:
  "This is a prototype clinical decision-support tool for demonstration purposes only.
  Do not use for clinical decision-making."
