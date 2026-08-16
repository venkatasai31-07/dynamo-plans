# Implementation Plan — dynamo-0a247ea-data-processing-and-etl

## Task Metadata
- **Task Name**: `dynamo/gps-segment-classifier`
- **Subcategory**: `Geospatial data processing`
- **Current PR**: PR #1 (open)
- **Git User ID**: `venkatasai31-07`
- **Local Path**: `C:\Users\HP\.gemini\antigravity-ide\scratch\dynamo-0a247ea-data-processing-and-etl`

---

## QC Review Soundness Findings & Proposed Fixes

### 1. Decisive Answer Discoverability (B5 & C3)
- **Problem**: Key algorithmic requirements were not explicitly stated in the instructions (e.g. stop segments radius/duration logic, segments merge rules, and distance/speed implied calculation). The author has since reverted/restored these specifics to `instruction.md` in the current branch.
- **Fix**: Verified that the instructions explicitly specify:
  - Average speed must be calculated using the Haversine distance and elapsed time between pings (and `reported_speed_kmh` is unused).
  - Stop segments are grouped by consecutive intervals where distance is within `stop_radius_m` and total elapsed duration is at least `stop_duration_min`.
  - Adjacent segments with the same classification must be merged.

### 2. Symlinked Output Path (E5)
- **Problem**: The verifier reads `/app/output/segments.csv` and `/app/output/summary.json` directly without checking if they are symbolic links, creating a security/exploit vector.
- **Fix**: Added `assert not os.path.islink(path)` guards to the file reading helpers inside `task/tests/test_outputs.py`.

---

## Detailed Step-by-Step Checklist

### Phase 1: Local Branch Setup
1. Checkout the `pr1-head` branch of `dynamo-0a247ea-data-processing-and-etl`.
2. Ensure `task/tests/reference.py` is restored.

### Phase 2: Harden Verifier
1. Open `task/tests/test_outputs.py` and modify `_read_segments()` and `_read_summary()` to check for symlinks.

### Phase 3: Local Testing and Verification
Verify that the task builds and the verifier passes using docker/harbor commands:
- **Build Docker Image**:
  `docker build -t gps-segment-task task/environment/`
- **Test using Harbor CLI**:
  `harbor task test --task .`
- **Directly using Docker**:
  `docker run --rm -it -v "$PWD/task/solution:/solution" -v "$PWD/task/tests:/tests" gps-segment-task bash -c "python3 /solution/solve.py && pytest /tests/test_outputs.py"`

### Phase 4: Push and Open PR
1. Commit all changes to the branch.
2. Push to GitHub fork and update/open a new PR.
