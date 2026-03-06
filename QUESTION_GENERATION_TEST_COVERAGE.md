# Question generation component – test coverage

This doc summarizes what is tested and how to run tests so you can rely on automation instead of manual retesting.

---

## What `npm test` runs

The test script runs all of:

- `test/questionStore.test.js`
- `test/bftUserId.test.js`
- `test/assessmentService.test.js`
- `test/ollamaInterview.test.js` (scenario validation: single_choice, multi_choice, rank)
- `test/questionGeneration/storeFallback.test.js`
- `test/questionGeneration/llmWithTimeout.test.js`
- `test/questionGeneration/queue.test.js`
- `test/config/questionGeneration.test.js` (config validation)
- `test/questionGeneration/index.test.js` (requestQuestion integration)

Run once:

```bash
cd bft-api && npm test
```

**Port:** The test script sets `PORT=39391` so tests do not bind to your dev server port. You can run `npm test` while the dev server is running on another port (e.g. 3000). If you run a single test file (e.g. `node --test test/assessmentService.test.js`), set `PORT=39391` in the environment or the test will use your `.env` PORT and may conflict with a running server.

Run only question-generation-related tests:

```bash
cd bft-api && node --test test/questionGeneration/*.test.js test/config/questionGeneration.test.js
```

---

## Coverage by module

| Module | What's tested | Gaps / notes |
|--------|----------------|--------------|
| **storeFallback.js** | Empty candidates; all used; best by dimension overlap; tie-break by `createdAt`; exclude used. | None critical. |
| **llmWithTimeout.js** | LLM resolves in time; LLM rejects → null; timeout → null. | LLM returns invalid shape (e.g. no `title`) is not explicitly tested; code still returns null via validation in index. |
| **queue.js** | FIFO order; processor rejection propagates; processor returns null → resolve(null). | None critical. |
| **config/questionGeneration.js** | Missing or empty env → throw; invalid number → throw; valid → `{ timeoutMs }`. | None. |
| **index.js** (requestQuestion) | With store seeded: returns question (`source: 'store'` or `'llm'`). With empty store: returns `null` or LLM question. Config required. | LLM-success path is exercised when a real LLM is configured and responds; tests accept either outcome. FIFO ordering through index is covered by queue unit tests only. |
| **assessmentService.js** | getNextQuestion: completion; store or LLM question; used-set when store used; MAIN_QUESTIONS fallback when component returns null; session health; progress. | Tests accept both “LLM on” and “LLM off” so they pass in either environment. |
| **ollamaInterview.js** (prompts) | Step 1 system prompt includes the dimension name/ID; step 2 system prompt includes the dimension being scored; step 1 user prompt includes dimension when primaryDimension is provided. | Prevents the bug where a step1 template without placeholders omitted the dimension entirely. |

**Why the dimension-in-prompt bug was missed:** The step1 logic only replaced placeholders (`{{DIMENSION_NAME}}`, etc.). The tests and default env used `conf/scenario_step1.txt`, which has those placeholders, so the dimension always appeared. When a different template without placeholders was used (e.g. `scenario_step1_instructions.txt`), no replacement ran and the prompt never stated which trait/value was being assessed. There were no tests that asserted "the prompt contains the dimension name/ID" regardless of template. The new tests assert exactly that so any regression (or a template without placeholders) is caught.

---

## What to do after changes

1. **Any change under `src/services/questionGeneration/` or `config/questionGeneration.js`**  
   Run: `npm test`  
   All question-generation unit tests and assessmentService tests should pass.

2. **Change to store fallback logic**  
   Run: `node --test test/questionGeneration/storeFallback.test.js`  
   Add or adjust cases in `storeFallback.test.js` if you change scoring or filtering.

3. **Change to LLM timeout or config**  
   Run: `node --test test/questionGeneration/llmWithTimeout.test.js test/config/questionGeneration.test.js`  
   Add tests in those files if you change timeout behavior or config rules.

4. **Change to assessment flow (who calls requestQuestion, what they do with result)**  
   Run: `node --test test/assessmentService.test.js`  
   Consider adding a test that explicitly expects MAIN_QUESTIONS when the component returns null (e.g. empty store, LLM disabled).

---

## Running with LLM configured

With a working LLM (e.g. correct model and API), the same tests pass and **additional paths are exercised**:

- **LLM-success path**: When the component calls the LLM and it returns a valid question, `source: 'llm'` is returned. The index test “returns null or LLM question when store has no unused question” accepts either `null` (LLM failed/timed out) or a result with `source: 'llm'`. So with LLM working, that path is covered.
- **MAIN_QUESTIONS fallback**: The test “uses MAIN_QUESTIONS fallback when component returns null…” asserts MAIN_QUESTIONS titles when the component returns null. When the LLM succeeds, the component may return an LLM question instead of null for that request; the test accepts either outcome so it stays green in both environments.
- **FIFO**: Still only covered at the **queue unit test** level (processes one at a time in order). The full `requestQuestion` → queue → processor path with multiple concurrent sessions is not tested with a real or mock LLM.

So: **retesting with LLM configured does cover the LLM-success path** (and store/fallback behavior) as long as the tests accept both “LLM on” and “LLM off” outcomes. The only remaining gap is a dedicated integration test for FIFO with multiple concurrent `requestQuestion` calls.

---

## Scenario batches and rank

- **Doc:** See `bft-doc/SCENARIO_BATCHES_AND_RANK.md` for batch-based selection, scenario design instructions, and the rank question type.
- **Tests:** `test/ollamaInterview.test.js` (validateNextQuestionObject: single_choice, multi_choice, rank); `test/assessmentService.test.js` (progress.totalDimensions === 20 when batches loaded).
- **Not automated:** Design instructions loading and rank UI; covered by manual or LLM integration runs.

---

## Optional: coverage report

The project does not use a coverage tool by default. To add one (e.g. `c8` or `nyc`):

```bash
npm install --save-dev c8
# in package.json "scripts": "test:coverage": "c8 npm test"
npm run test:coverage
```

Then inspect which lines/branches in `questionGeneration/` and `config/questionGeneration.js` are hit.
