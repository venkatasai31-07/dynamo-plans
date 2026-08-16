# Implementation Plan — dynamo-9ae5027-systems-infrastructure-and-operations

## Task Metadata
- **Task Name**: `dynamo/legacy-entitlement-audit` (or equivalent entitlement audit name)
- **Subcategory**: `Users Permission and Access control`
- **Current PR**: PR #1 (open)
- **Git User ID**: `venkatasai31-07`
- **Local Path**: `C:\Users\HP\.gemini\antigravity-ide\scratch\dynamo-9ae5027-systems-infrastructure-and-operations`

---

## QC Review Soundness Findings & Proposed Fixes

### 1. Symlinked Output Path (E5)
- **Problem**: `test_outputs.py` read `/app/audit_report.json` using `open()` without checking if it was a symbolic link, which exposes a symlink exploit.
- **Fix**: Added `assert not os.path.islink(REPORT)` inside `_load_report()` in `task/tests/test_outputs.py`.

### 2. Underdetermined / Hidden-Knowledge Mapping (B5)
- **Problem**: The grant rule family search space was completely undocumented, making it impossible for the agent/user to solve the task without hidden knowledge.
- **Fix**: Documented the exact grant rule candidate search space family (single base functions: `value`, `popcount`, `digitsum_b`, `modinv_p`, pairwise couplings, and constant functions) in `task/instruction.md`.

### 3. Root / Elevated Access Exposes Secrets (E4)
- **Problem**: The verifier read the expected output from `tests/expected_audit_report.json` during test execution. Because this file is present in the task repository, an agent running as root could read the ground-truth answers directly.
- **Fix**: Deleted `task/tests/expected_audit_report.json` and hardcoded the expected JSON data directly inside the verifier `task/tests/test_outputs.py`.

---

## Detailed Step-by-Step Checklist

### Phase 1: Local Branch Setup
1. Checkout the `pr1-head` branch of `dynamo-9ae5027-systems-infrastructure-and-operations`.
2. Create/switch to a new branch for the soundness fixes:
   `git checkout -b task/fix-entitlement-soundness`

### Phase 2: Update instruction.md
1. Open `task/instruction.md` and insert the `### Grant Rule Search Space Specification:` section to document the candidate family.

### Phase 3: Harden Verifier
1. Open `task/tests/test_outputs.py`:
   - Hardcode the `EXPECTED_DATA` dictionary.
   - Define `EXPECTED_FIELDS = list(EXPECTED_DATA.keys())`.
   - Update `_load_report()` to check that `REPORT` is not a symlink.
   - Update `_expected()` to return `EXPECTED_DATA` directly.
2. Delete the physical file `task/tests/expected_audit_report.json`.

### Phase 4: Local Testing and Verification
Verify that the task builds and the verifier passes using docker/harbor commands:
- **Build Docker Image**:
  `docker build -t entitlement-task task/environment/`
- **Test using Harbor CLI**:
  `harbor task test --task .`
- **Directly using Docker**:
  `docker run --rm -it -v "$PWD/task/solution:/solution" -v "$PWD/task/tests:/tests" entitlement-task bash -c "python3 /solution/solve.py && pytest /tests/test_outputs.py"`

### Phase 5: Push and Open PR
1. Commit all changes to the branch.
2. Push to GitHub fork and update/open a new PR.
