# Implementation Plan — dynamo-02af1ac-data-processing-and-etl

## Task Metadata
- **Task Name**: `dynamo/validate-event-timestamps`
- **Subcategory**: `Data validation`
- **Current PR**: PR #3 (open)
- **Git User ID**: `venkatasai31-07`
- **Local Path**: `C:\Users\HP\.gemini\antigravity-ide\scratch\dynamo-02af1ac-data-processing-and-etl`

---

## QC Review Soundness Findings & Proposed Fixes

### 1. Pass@2 Difficulty Failure (Pre-check failed due to being too easy)
- **Problem**: The dataset was too small and simple. Naive agents solved it easily without implementing fold-awareness and timezone verification robustly.
- **Fix**: Re-generated the `events.csv` dataset with:
  - Higher source count (`NUM_SOURCES = 95`) and event density (`EVENTS_PER_SOURCE = (300, 500)`), yielding over 53,000+ total rows.
  - Increased density of transition-window edge cases: spring-forward gap-hour rows, fall-back ambiguous fold=1 (non-default) rows, and near-miss UTCs (off by 1-5 minutes).
  - This forces agents to execute fold-symmetric validation, timezone consistency votes, and fast O(N) log verification to pass.

---

## Detailed Step-by-Step Checklist

### Phase 1: Local Branch Setup
1. Checkout the `submission` branch of `dynamo-02af1ac-data-processing-and-etl`.
2. Ensure all changes are staged/committed.

### Phase 2: Execute Generator
1. Run `python task/solution/generate_events.py` to rebuild the larger `events.csv` dataset.
2. Copy the new `events.csv` to both:
   - `task/environment/data/events.csv`
   - `task/tests/data/events.csv`

### Phase 3: Local Testing and Verification
Verify that the task builds and the verifier passes using docker/harbor commands:
- **Build Docker Image**:
  `docker build -t validate-timestamp-task task/environment/`
- **Test using Harbor CLI**:
  `harbor task test --task .`
- **Directly using Docker**:
  `docker run --rm -it -v "$PWD/task/solution:/solution" -v "$PWD/task/tests:/tests" validate-timestamp-task bash -c "python3 /solution/solve.py && pytest /tests/test_outputs.py"`

### Phase 4: Push and Open PR
1. Commit all changes to the branch.
2. Push to GitHub fork and update the PR to trigger the automated QC gate.
