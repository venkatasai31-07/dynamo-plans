# Implementation Plan — dynamo-a83e4bc-machine-learning-and-ai

## Task Metadata
- **Task Name**: `dynamo/monocular-trajectory-sfm` (or equivalent trajectory reconstruction name)
- **Subcategory**: `Computer vision and SfM`
- **Current PR**: PR #1 (open)
- **Git User ID**: `venkatasai31-07`
- **Local Path**: `C:\Users\HP\.gemini\antigravity-ide\scratch\dynamo-a83e4bc-machine-learning-and-ai`

---

## QC Review Soundness Findings & Proposed Fixes

### 1. Narrow / Hardcodable Held-Out Coverage (C3)
- **Problem**: `test_min_reconstructed_point_count` only verified that the raw row count of `points3d.csv` was at least 150. A mutant or lazy submission could output 200 copies of a single point and pass the check.
- **Fix**: Update `test_min_reconstructed_point_count` in `task/tests/test_outputs.py` to parse the 3D points, compute the number of *distinct* (unique) points, and assert that it is at least 150:
  ```python
  points = np.array([[float(v) for v in row] for row in points_raw], dtype=float)
  unique_points = np.unique(points, axis=0)
  assert len(unique_points) >= MIN_POINTS, \
      f"points3d.csv has only {len(unique_points)} unique 3D points, need >= {MIN_POINTS}"
  ```

### 2. Undocumented Accuracy Threshold Requirements (B4)
- **Problem**: The verifier enforces six numerical accuracy thresholds that were never mentioned in the instructions, leaving the user/agent without targets.
- **Fix**: Updated `task/instruction.md` to add a new section explicitly documenting all six acceptance thresholds:
  - **Absolute Trajectory Error (ATE)**: mean error < 1.2%, maximum error < 2.0%
  - **Relative Pose Error (RPE)**: translation mean error < 0.35%, rotation mean error < 0.40 degrees, rotation maximum error < 1.1 degrees
  - **Point Cloud Precision**: mean distance to ground-truth surfaces < 2.5%
  - **Minimum Point Count**: at least 150 distinct 3D points.

---

## Detailed Step-by-Step Checklist

### Phase 1: Local Branch Setup
1. Checkout the `pr1-head` branch of `dynamo-a83e4bc-machine-learning-and-ai`.
2. Create/switch to a new branch for the soundness fixes:
   `git checkout -b task/fix-sfm-soundness`

### Phase 2: Update instruction.md
1. Open `task/instruction.md` and add the `### Reconstruction Accuracy Requirements:` section.

### Phase 3: Harden Verifier
1. Open `task/tests/test_outputs.py` and modify `test_min_reconstructed_point_count` to enforce unique points checking.

### Phase 4: Local Testing and Verification
Verify that the task builds and the verifier passes using docker/harbor commands:
- **Build Docker Image**:
  `docker build -t sfm-task task/environment/`
- **Test using Harbor CLI**:
  `harbor task test --task .`
- **Directly using Docker**:
  `docker run --rm -it -v "$PWD/task/solution:/solution" -v "$PWD/task/tests:/tests" sfm-task bash -c "python3 /solution/solve.py && pytest /tests/test_outputs.py"`

### Phase 5: Push and Open PR
1. Commit all changes to the branch.
2. Push to GitHub fork and update/open a new PR.
