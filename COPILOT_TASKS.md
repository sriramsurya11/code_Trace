# CodeTrace: Code Visualizer for DSA

Paste code (Python, Java, C, C++, JavaScript), run it for real, record every step, and draw it as a visualization (recursion tree, DP table, linked list, graph) with a step-through slider, like a debugger with pictures.

## 0. Instructions for GitHub Copilot (read first)

- Work on ONE task at a time, in the order below. Do not skip ahead.
- After each task: run the tests, fix failures, then commit with message `task-<id>: <title>`.
- Never mark a task done unless its **Acceptance** checks pass.
- Do not simulate code execution with an LLM. Traces MUST come from really running the code.
- Never run user code outside the sandbox (see Task 3.x). No `exec`/`eval` of user code in the API process.
- Keep the trace format in section 2 as the single contract. If it must change, update section 2 and the JSON schema first, then the code.
- If something in this plan is unclear or impossible, write the question in `docs/OPEN_QUESTIONS.md` and continue with the safest option.

## 1. Tech stack and repo layout

- Frontend: React + Vite + TypeScript, Tailwind CSS, D3 (or React Flow) for trees/graphs, Monaco editor for code input.
- Backend API: Python 3.11 + FastAPI.
- Runners: one Docker image per language. Each runner takes source code, runs it with a tracer, prints trace JSON to stdout.
- Tests: pytest (backend + tracers), Vitest + React Testing Library (frontend), Playwright (end to end).

```
codetrace/
  docs/
    TRACE_FORMAT.md        # copy of section 2
    trace.schema.json      # JSON Schema of the trace
    OPEN_QUESTIONS.md
  frontend/                # React app
  backend/                 # FastAPI app
    app/
    tests/
  runners/
    python/                # Dockerfile + tracer.py
    java/                  # Dockerfile + tracer (JDI or instrumentation)
    javascript/
    c_cpp/
  fixtures/                # sample programs + expected "golden" traces
    python/ java/ javascript/ c/ cpp/
  e2e/                     # Playwright tests
  docker-compose.yml
  README.md
```

## 2. Trace format (the contract)

A trace is one JSON object:

```json
{
  "version": 1,
  "language": "java",
  "status": "ok",
  "error": null,
  "truncated": false,
  "stdout": "",
  "events": [ ]
}
```

- `status`: `ok` | `runtime_error` | `timeout` | `compile_error` | `limit_exceeded`
- `truncated`: true if the event limit was hit (see limits below)

Every event has `step` (0,1,2,... increasing), `type`, and `line` (1-based, source line):

| type | extra fields |
|---|---|
| `call` | `frameId`, `parentFrameId` (null for first), `fn`, `args` (name -> value), `depth` |
| `line` | `frameId`, `locals` (name -> value) |
| `return` | `frameId`, `value` |
| `assign` | `frameId`, `name`, `old`, `new` (optional, can be derived from `line` events) |
| `exception` | `frameId`, `message` |

**Value encoding** (so the frontend can detect data structures):

```json
{"t":"int","v":3}
{"t":"str","v":"(()"}
{"t":"bool","v":true}
{"t":"null"}
{"t":"list","v":[ ...values ]}            // arrays, lists, vectors
{"t":"map","v":[[key,value], ...]}        // maps, dicts
{"t":"obj","id":"o12","cls":"ListNode","fields":{"val":{...},"next":{"t":"ref","id":"o13"}}}
{"t":"ref","id":"o13"}                    // pointer to an already-emitted object (handles cycles)
{"t":"trunc"}                             // value too large, cut
```

**Limits (enforced by every tracer):**
- max 20,000 events, then stop and set `truncated: true`
- max recursion depth shown: 200
- max string length 200 chars, max list items 100 per value
- max runtime 5 seconds, max memory 256 MB

## 3. Tasks

### Phase 0: Spec and scaffolding

**Task 0.1: Repo scaffold**
- Create the folder layout above, README with setup steps, `.gitignore`, `docker-compose.yml` (frontend, backend).
- Acceptance: `docker compose up` starts frontend on :5173 and backend on :8000; `GET /health` returns `{"ok":true}`.

**Task 0.2: Trace schema**
- Write `docs/TRACE_FORMAT.md` and `docs/trace.schema.json` from section 2.
- Add a backend function `validate_trace(trace)` that validates against the schema.
- Acceptance: pytest tests with 3 valid and 3 invalid sample traces pass.

**Task 0.3: Golden fixtures**
- Create `fixtures/python/` and `fixtures/java/` with these programs (one file each): 
  1. `generate_parentheses` (the BUGGY version below, so the tool must make the bug visible)
  2. `generate_parentheses_fixed`
  3. `fibonacci_memo` (DP)
  4. `reverse_linked_list`
  5. `dfs_graph`
- Buggy Java version:
```java
class Solution {
    List<String> res = new ArrayList<>();
    public List<String> generateParenthesis(int n) {
        backtrack(n, n, "");
        return res;
    }
    public void backtrack(int open, int close, String paren) {
        if (open == 0 && close == 0) { res.add(paren); return; }
        if (open > 0) backtrack(open - 1, close, paren + "(");
        if (close > 0) backtrack(open, close - 1, paren + ")");
    }
}
```
- Fixed version changes the second condition to `if (close > open)`.
- Acceptance: for n=2 the buggy version gives 3 results with one invalid (`"())("`), the fixed gives exactly `["(())","()()"]`. Verify by actually running them.

### Phase 1: Python tracer (first end-to-end slice)

**Task 1.1: Python tracer**
- `runners/python/tracer.py` uses `sys.settrace` to emit `call`, `line`, `return`, `exception` events in the section 2 format.
- Only trace functions defined in the user file (skip stdlib).
- Serialize values with the encoding in section 2, handle cycles with `ref`, and enforce all limits.
- Entry point: reads source from a file, the call to run (e.g. `Solution().generateParenthesis(2)`) from a CLI arg, prints trace JSON to stdout.
- Acceptance: golden trace for `generate_parentheses` (n=2) has the expected number of `call` events and correct args at each call; trace passes `validate_trace`.

**Task 1.2: Entry-point detection**
- Backend helper that finds the function to call: user can give an explicit call, otherwise auto-detect the class/function and ask for inputs.
- Acceptance: unit tests for LeetCode-style `class Solution` with one public method, and for plain top-level functions.

**Task 1.3: Tracer tests**
- Tests: recursion depth limit, event limit (infinite loop gets `limit_exceeded`), exception inside user code (`runtime_error` with the line number), print output captured in `stdout`.
- Acceptance: all pass.

### Phase 2: Backend API (local, unsandboxed only for development)

**Task 2.1: `/api/run` endpoint**
- `POST /api/run` body: `{language, code, entry, input}` -> returns trace JSON.
- For now call the Python tracer in a subprocess with a timeout. Mark this path `DEV_ONLY` and disable it unless env `ALLOW_UNSANDBOXED=1`.
- Acceptance: endpoint returns a valid trace for the python fixtures; returns `timeout` for `while True: pass`.

### Phase 3: Sandbox (do this before exposing anything publicly)

**Task 3.1: Docker runner**
- Backend runs each request in a fresh container from the language's image with: `--network none`, `--memory 256m`, `--cpus 0.5`, `--pids-limit 64`, read-only filesystem except a tmpfs for the code, non-root user, hard wall-clock timeout (kill container).
- Code is passed via stdin or a mounted temp file, never built into a shell string.
- Acceptance: tests where user code (a) tries to open a network connection, (b) fork-bombs, (c) allocates huge memory, (d) reads `/etc/passwd`-style host files, (e) sleeps forever. All must fail safely and the API stays up.

**Task 3.2: Rate limiting and size limits**
- Max code size 20 KB, max 10 requests/min per IP, concurrency cap on containers.
- Acceptance: tests for oversize code (413) and burst requests (429).

### Phase 4: Frontend with recursion tree

**Task 4.1: Editor page**
- Monaco editor, language dropdown, entry/input field, Run button, loading and error states (compile error, runtime error, timeout shown clearly with line highlight).
- Acceptance: component tests; running the buggy fixture shows a trace in app state.

**Task 4.2: Trace to tree**
- Pure function `buildCallTree(events)` -> tree of `{frameId, fn, args, returnValue, children, firstStep, lastStep}`.
- Acceptance: Vitest tests: buggy n=2 tree has the expected number of nodes and parent/child links.

**Task 4.3: Recursion tree view**
- Draw the tree with D3 or React Flow: each node shows function name, args, and return value; edges are labeled by the branch taken when it can be inferred from the line.
- Step slider + Play/Pause/Prev/Next/speed. At step N: nodes not yet called are hidden/greyed, the current call is highlighted, the active call stack is highlighted, finished calls show their return value.
- Side panel: current variable values (locals of current frame) and the source with the current line highlighted.
- Backtracking support: highlight leaf nodes that reach the base case, show the `res` list growing when the program mutates it.
- Acceptance: Playwright test loads the buggy fixture and steps to the end; the tree contains the invalid result path and the screenshot is saved to `e2e/screenshots/`.

**Task 4.4: Bug hint (non-LLM)**
- Show a simple "result summary": list of results found. Add a "compare with expected" input where the user can paste the expected output; highlight which result is extra/missing and which tree leaf produced it.
- Acceptance: for the buggy fixture with expected `["(())","()()"]`, the extra result `"())("` is flagged and its leaf node is highlighted.

### Phase 5: More topics (renderers use the SAME trace)

**Task 5.1: Renderer plugin system**
- A `Renderer` interface: `canRender(trace): score`, `render(trace, step)`. The app lists applicable renderers as tabs (e.g. Call Tree, Variables).
- Acceptance: unit tests for scoring; the recursion renderer is moved onto this interface with no regression.

**Task 5.2: Array / DP table renderer**
- Detect 1D/2D list variables that change over time; draw them as cells; highlight the cell changed at this step; show previous values faded.
- Acceptance: `fibonacci_memo` and one 2D DP fixture (LCS) animate correctly; cell-change highlighting verified by component test.

**Task 5.3: Linked list renderer**
- Detect objects with a `next` field; draw nodes and arrows; show pointer variables (`prev`, `curr`) as labels on nodes; support cycles.
- Acceptance: `reverse_linked_list` shows the arrows flipping step by step.

**Task 5.4: Graph renderer**
- Detect adjacency list/matrix input; draw nodes and edges; highlight visited nodes and the current node, show stack/queue contents when present.
- Acceptance: `dfs_graph` shows visit order.

### Phase 6: More languages

Add one at a time, each with its own Docker image, tracer, fixtures and tests. Each must pass the same golden fixtures as Python (same logical call tree for `generate_parentheses`).

**Task 6.1: Java tracer**
- Use the Java Debug Interface (JDI): launch the user's compiled class in a debuggee JVM, set method entry/exit requests and step requests, read arguments and locals (compile with `-g`).
- Only trace methods in user classes. Enforce limits from section 2.
- Acceptance: golden trace for the Java buggy fixture matches the Python one in call count and args (string values compared as text).

**Task 6.2: JavaScript tracer**
- Instrument the source with an AST tool (Babel or acorn) to emit call/return/line events; run in Node inside the container.
- Acceptance: golden fixtures pass.

**Task 6.3: C/C++ tracer**
- Compile with `-g -O0`; drive GDB (Python API or MI) to step through and read frames and locals. Pointer-based structures (linked list) are best effort.
- If this proves too unreliable, document limits in `docs/OPEN_QUESTIONS.md` and support only recursion + arrays for C/C++.
- Acceptance: recursion fixtures pass for C and C++; compile errors are reported with line numbers.

### Phase 7: Quality, polish, deploy

**Task 7.1: Test suite and CI**
- GitHub Actions: lint, backend tests, frontend tests, build Docker images, Playwright e2e on the buggy fixture.
- Acceptance: CI green on main.

**Task 7.2: UX polish**
- Share link for a trace (store code + trace by id), dark mode, keyboard shortcuts (left/right arrows to step), mobile-friendly layout, sample programs dropdown from `fixtures/`.
- Acceptance: Playwright tests for share link and keyboard stepping.

**Task 7.3: Docs and deploy**
- README with architecture diagram, how to add a language, how to add a renderer. Deploy frontend (static host) and backend + runners on a VM with Docker. No secrets in the repo.
- Acceptance: a fresh clone follows the README and works.

### Optional: AI explanations (only after everything above works)

- Button "Explain this trace": send the trace summary and code to an LLM to explain what went wrong in plain language. Clearly label it as AI explanation. It must NEVER be the source of the trace itself.

## 4. Definition of done (whole project)

1. Pasting the buggy `generateParenthesis` Java code and running it shows the call tree where the extra `")"` branch at `(open=1, close=1, "()")` is visible and the invalid result is flagged.
2. At least Python and Java fully supported for recursion/backtracking; DP, linked list and graph renderers working with Python at minimum.
3. Sandbox tests pass; no user code ever runs outside a container.
4. All tests (unit, integration, e2e) pass in CI.
5. README lets a new developer run it in under 10 minutes.
