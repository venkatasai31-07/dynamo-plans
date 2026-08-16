# Implementation Plan — dynamo-005c675-systems-infrastructure-and-operations

## Task Metadata
- **Task Name**: `dynamo/generation-supervisor`
- **Subcategory**: `OS process and service management`
- **Current PR**: PR #1 (closed/to be updated/re-opened via new PR or branch update)
- **Git User ID**: `venkatasai31-07`
- **Local Path**: `C:\Users\HP\.gemini\antigravity-ide\scratch\dynamo-005c675-systems-infrastructure-and-operations`

---

## QC Review Soundness Findings & Proposed Fixes

### 1. Underdetermined Mapping & Tie-Breaking Order (QC Findings 1 & 2)
- **Problem**: The exact topological sorting and dependency-stopping order was underdetermined. Kahn's topological sort and standard DFS-based topological sort (tie-broken by declaration order) can produce different valid topological orders, leading to divergent behavior on scenarios like:
  - `services: root, web(->root, api), solo(->root), api(->root)`.
  - When `root` times out, the DFS topo sort stops dependents in order `[api, web, solo]`, whereas other SPEC-compliant sorting algorithms could stop them in a different order.
- **Fix**: Update `instruction.md` to explicitly define the exact DFS-based topological sort algorithm used by the supervisor engine:
  - Topological sorting is done using a depth-first search (DFS) visiting services in declaration order and dependencies in declared order (left-to-right).
  - Dependents are stopped in the reverse order of their appearance in the DFS-ordered list.

### 2. Narrow / Hardcodable Held-Out Coverage (QC Finding 3)
- **Problem**: The existing test suite was too narrow to detect if the engine implemented a wrong tie-break topological sorting order.
- **Fix**: Added a new scenario `topo-tie-break.trace` and its expected output `topo-tie-break.json` to the test suite:
  - The scenario implements the exact services and timeout event described above.
  - Added `'topo-tie-break'` to the `CASES` tuple in `task/tests/test_outputs.py`.

### 3. Undisclosed Verifier Static Token Assertions (Pass@2 Difficulty Suggestion)
- **Problem**: The verifier statically asserts (via AST inspection) that specific helper method token strings appear inside the bodies of the hazard-localized handler functions (e.g. `_maybe_shutdown_complete` inside `_handle_shutdown`, and `_signal_generation` inside both `_handle_exec_fail` and `_handle_adopt`). This undisclosed requirement caused correct solutions to fail the verifier.
- **Fix**: Disclosed these helper method token requirements explicitly in `instruction.md` to guide the agent.

---

## Detailed Step-by-Step Checklist

### Phase 1: Local Branch Setup
1. Checkout the `pr1-head` branch of `dynamo-005c675-systems-infrastructure-and-operations`.
2. Create/switch to a new branch for the soundness fixes:
   `git checkout -b task/fix-supervisor-soundness`

### Phase 2: Create Topo Tie-Break Scenario
1. Write the trace scenario to `task/tests/scenarios/topo-tie-break.trace`.
2. Run the supervisor package locally using the oracle code to generate the expected JSON output at `task/tests/expected/topo-tie-break.json`:
   `$env:PYTHONPATH="task/environment/app"; python -m src.cli --scenario task/tests/scenarios/topo-tie-break.trace --output task/tests/expected/topo-tie-break.json`

### Phase 3: Update Verifier and Instruction
1. Add `'topo-tie-break'` to `CASES` in `task/tests/test_outputs.py`.
2. Update `task/instruction.md` to document the DFS topological sort tie-breaking rules and the required static helper method calls.

### Phase 4: Local Testing and Verification
Verify that the task builds and the verifier passes using docker/harbor commands:
- **Build Docker Image**:
  `docker build -t supervisor-task task/environment/`
- **Test using Harbor CLI**:
  `harbor task test --task .`
- **Directly using Docker** (to ensure the verifier rejects wrong implementations and accepts the oracle):
  `docker run --rm -it -v "$PWD/task/solution:/solution" -v "$PWD/task/tests:/tests" supervisor-task bash -c "bash /solution/solve.sh && pytest /tests/test_outputs.py"`

### Phase 5: Push and Open PR
1. Commit all changes to the branch.
2. Push to GitHub fork and open a new PR.
