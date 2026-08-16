# Implementation Plan — dynamo-e657d04-systems-infrastructure-and-operations

## Task Metadata
- **Task Name**: `dynamo/wal-crash-recovery`
- **Subcategory**: `Logging monitoring and observability`
- **Current PR**: PR #1 (open)
- **Git User ID**: `venkatasai31-07`
- **Local Path**: `C:\Users\HP\.gemini\antigravity-ide\scratch\dynamo-e657d04-systems-infrastructure-and-operations`

---

## QC Review Soundness Findings & Proposed Fixes

### 1. Underdetermined Mapping & Narrow Coverage (QC Findings 1 & 2)
- **Problem**: The verifier only had the original 19 alerts where no `memory_pressure` window check was failed due to time limits. A mutant that disables the 30-second window check (e.g. changing the window threshold to 100000s) still passes all tests.
- **Fix**: Modify the binary `metrics.wal` file at authoring/setup time to include three new records for a new `source_id = 8` in Shard 0:
  - Sample 1: value `93.0` at timestamp `1700000030`
  - Sample 2: value `94.0` at timestamp `1700000075` (45s gap, must NOT trigger alert)
  - Sample 3: value `95.0` at timestamp `1700000085` (10s gap, MUST trigger alert)
- This generates two new alerts in `alerts.json` (a `high_cpu` and a `memory_pressure` alert at `1700000085` for source 8), which are added to the verifier's `EXPECTED_ALERTS` in `test_outputs.py`. A mutant that fails to enforce the 30s window check will fire a memory_pressure alert at `1700000075` and fail verification.

### 2. Ambiguity & Missing Specifications (QC Findings 3, 4, & 5)
- **Problem**: The instruction did not explicitly define:
  1. The chronological sorting requirement before delta decoding.
  2. The deduplication selection policy.
- **Fix**: Update `instruction.md` to:
  - Clarify that delta-encoded values must be sorted chronologically by timestamp within each shard before accumulation.
  - Document that the first occurrence of duplicate samples (same `source_id` and `timestamp`) must be kept, and subsequent duplicates discarded during shard-order (ascending by shard ID) and chronological processing.

---

## Detailed Step-by-Step Checklist

### Phase 1: Local Branch Setup
1. Checkout the `pr1-head` branch of `dynamo-e657d04-systems-infrastructure-and-operations`.
2. Create/switch to a new branch for soundness fixes:
   `git checkout -b task/fix-wal-soundness`

### Phase 2: Modify metrics.wal & Verifier
1. Replace `task/environment/data/metrics.wal` with the regenerated binary file containing the custom source 8 records.
2. Update `EXPECTED_ALERTS` in `task/tests/test_outputs.py` to insert the new source 8 alerts at trigger timestamp `1700000085`.

### Phase 3: Update instruction.md
1. Edit `task/instruction.md` to:
   - Explicitly define chronological sorting order for delta-decoding.
   - Add a `Deduplication` policy section to define row-selection rules.

### Phase 4: Local Testing and Verification
Verify that the task builds and the verifier passes using docker/harbor commands:
- **Build Docker Image**:
  `docker build -t wal-task task/environment/`
- **Test using Harbor CLI**:
  `harbor task test --task .`
- **Directly using Docker** (to ensure the verifier rejects wrong implementations and accepts the oracle):
  `docker run --rm -it -v "$PWD/task/solution:/solution" -v "$PWD/task/tests:/tests" wal-task bash -c "bash /solution/solve.sh && pytest /tests/test_outputs.py"`

### Phase 5: Push and Update PR
1. Commit all changes to the branch.
2. Push to GitHub fork and update the PR.
