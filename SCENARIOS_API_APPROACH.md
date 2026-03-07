# Current approach: config, startup, and scenario prompts

This document describes how the bft-api is configured, how startup validation works, and how scenario-based question generation and interview tailoring work. It reflects the current code and conventions (no defaults in code, no path fallbacks, fail fast at startup).

---

## 1. Config and startup

### 1.1 Principles

- **No defaults in code.** Required values are not hardcoded. They must be set in `.env` (see `bft-api/env.example`). Missing or invalid required config causes the process to exit immediately with a clear message.
- **Single .env.** The API loads exactly one env file: `bft-api/.env`. The path is resolved from the API project directory (`config/index.js`, `config/llm.js` use `path.join(__dirname, '..', '.env')`), so the server behaves the same regardless of current working directory.
- **No path fallbacks.** If a required file path is set in `.env`, that path must exist. There is no silent fallback to another path (e.g. no `conf/legacy/` fallback). Invalid paths cause startup to exit.

### 1.2 Load and validate order

On startup (`server.js`):

1. **config/index.js** – Loads `.env` from bft-api, then validates: `PORT`, `NODE_ENV`, `CORS_ORIGIN`, `BFT_QUESTIONS_STORE_DIR`. Exits on failure.
2. **config/llm.js** – Loads `.env` again from bft-api (idempotent), validates all required LLM and report-prompt env vars and that required prompt files exist. Exits on failure.
3. **config/assessment.js** – Validates pregen/interview env (e.g. `BFT_PREGEN_QUEUE_CAP`, `MIN_SIGNAL_PER_DIMENSION`). Exits on failure.
4. **runStartupChecks(config)** – Ensures `BFT_QUESTIONS_STORE_DIR` is writable, required data files exist, **BFT_SCENARIO_STEP1_INSTRUCTIONS_FILE** and **BFT_SCENARIO_STEP3_INSTRUCTIONS_FILE** are set and point to existing files, then runs optional LLM connectivity check (with timeout). Exits on failure.

Required scenario prompt env vars validated at startup:

- `BFT_SCENARIO_STEP1_INSTRUCTIONS_FILE` (e.g. `conf/scenario_step1.txt`)
- `BFT_SCENARIO_STEP3_INSTRUCTIONS_FILE` (e.g. `conf/scenario_step3.txt`)

Step 2 prompt files (`scenario_step2_critique.txt`, `scenario_step2_judge.txt`) are not env-configured; they are fixed under `conf/` and have in-memory fallbacks if missing. Startup does not validate them.

---

## 2. Scenario source (store vs generate)

**BFT_SCENARIO_STORE_FIRST** (required, true/false) controls where to get the next scenario first:

- **`true`**: Try the question store first. If a suitable unused question exists for the profile and dimension set, return it. Otherwise call the LLM, then (if LLM fails) try the store again.
- **`false`**: Try the LLM first. If it fails or times out, fall back to the store.

Config is validated when the question generation component is first used (no default in code; see `bft-api/env.example`).

---

## 3. Scenario question generation (three-step flow)

Generation of one scenario-based interview question uses a fixed three-step flow in `src/lib/ollamaInterview.js`.

### 3.1 Step 1: Plain-text scenario

- **Purpose:** Generate one plain-text scenario (title, situation, options) that probes the primary dimension without telegraphing it.
- **Config:** `BFT_SCENARIO_STEP1_INSTRUCTIONS_FILE` (required; validated at startup).
- **Prompt:** System = step1 instructions + dimension context + **tailoring block**. User = asked titles, previous answers, dimension hints, optional batch theme / dilemma anchor.
- **Output:** Parsed `{ title, situation, options }`. If parse fails, the step is retried (up to `MAX_CRITIQUE_ATTEMPTS`).

### 3.2 Step 2: Critique and judge

- **Purpose:** Check that the scenario does not give away the dimension. Two LLM calls: (1) critique prompt produces one sentence; (2) judge prompt (PASS/FAIL) on that sentence.
- **Config:** Fixed paths in code: `conf/scenario_step2_critique.txt`, `conf/scenario_step2_judge.txt`. If either file is missing, in-memory fallback templates are used. Not in `.env`; not validated at startup.
- **Output:** Boolean (pass/fail). If fail, step 1 is retried (up to `MAX_CRITIQUE_ATTEMPTS`). If all attempts fail, no question is returned.

### 3.3 Step 3: Format and score

- **Purpose:** Turn the plain-text scenario into the final JSON question (title, description, type, options with `dimensionScores`) and derive `dimensionSet`.
- **Config:** `BFT_SCENARIO_STEP3_INSTRUCTIONS_FILE` (required; validated at startup).
- **Output:** `{ question, dimensionSet }`. Validated (ids, option shape, primary dimension scores in range). If invalid, null is returned.

---

## 4. Interview tailoring block

The pre-survey profile is used to build a short **tailoring block** appended to the **step 1** system prompt only. It does not apply to step 2 or step 3.

### 4.1 Contents (in order)

- **AUDIENCE:** From demographics (e.g. age group, gender) when present.
- **STYLE:** From dominant and secondary personality clusters (primarily X; also Y; blend when possible). Uses `short` labels from `src/data/personality_clusters.json`.
- **TONE:** From `toneInstruction` (pre-survey Q5) if set; otherwise from `secondaryTone` mapped to cluster short.
- **COMPLEXITY:** From `complexityInstruction` (derived from age group / Q2) when set.

There is **no AVOID line**. Previously, an "AVOID: ... Do not use these framings" line was built from `avoidClusters` and cluster `avoid` phrases; that was removed so it does not contradict the scenario generation instructions. The profile still stores `avoidClusters` for possible future use; it is not sent to the LLM.

### 4.2 Data source

- **Profile:** `preSurveyProfile` (dominant, secondary, secondaryTone, demographics, toneInstruction, complexityInstruction). Comes from the UI pre-survey and is sent with session create.
- **Clusters:** `src/data/personality_clusters.json` (name, short, avoid). Only `short` is used in the tailoring block; `avoid` is not used in any prompt.

---

## 5. Prompt file layout (conf/)

| File | How configured | Validated at startup |
|------|----------------|----------------------|
| scenario_step1.txt | `BFT_SCENARIO_STEP1_INSTRUCTIONS_FILE` | Yes; must exist |
| scenario_step2_critique.txt | Hardcoded in code | No; fallback if missing |
| scenario_step2_judge.txt | Hardcoded in code | No; fallback if missing |
| scenario_step3.txt | `BFT_SCENARIO_STEP3_INSTRUCTIONS_FILE` | Yes; must exist |
| report_profile_system_prompt.txt | `LLM_REPORT_PROFILE_SYSTEM_PROMPT_FILE` | Yes (required by config/llm) |
| report_hybrid_system_prompt.txt | `LLM_REPORT_HYBRID_SYSTEM_PROMPT_FILE` | Yes |
| report_recommendations_system_prompt.txt | `LLM_REPORT_RECOMMENDATIONS_SYSTEM_PROMPT_FILE` | Yes |

Paths in `.env` are relative to the bft-api project root. No fallback to `conf/legacy/` or any other path.

---

## 6. References

- **Config contracts:** `bft-doc/CONTRACTS.md` (Config, Ollama interview).
- **Architecture:** `bft-doc/ARCHITECTURE.md`.
- **Workspace rules:** `.cursor/rules/no-defaults-in-code.mdc` (no defaults, fail fast, document in config).
- **Env template:** `bft-api/env.example`.
