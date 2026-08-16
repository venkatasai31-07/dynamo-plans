# Master PR Implementation & Quality Plan (Project Dynamo)

**Author / Git User ID:** `venkatasai31-07`  
**Workspace Root:** `C:\Users\HP\.gemini\antigravity-ide\scratch\`  
**Target Platform:** Project Dynamo Benchmark Tasks  
**Goal:** Pass all QA/QC review gates, Pass@2 difficulty tests, and prevent any soundness bypasses.

---

## Part 1: Playbook Quality Strategy & Principles
Based on the Project Dynamo difficulty and design playbook, all repositories are hardened against three main failure modes:
1. **Underdetermined Gaps (B5 / C3)**: Stating domain facts without pre-digested instructions, ensuring the correct logical mapping is mathematically and logically entailed rather than arbitrary or hidden.
2. **Symlink / Environment Exploit Path (E5)**: Hardening verifiers to assert that generated output files are not symbolic links.
3. **Secrets Exposure / Trivial Bypass (E4)**: Removing pre-computed ground-truth answers from the repository to prevent root-privileged agent discovery.

---

## Part 2: Task-by-Task Implementation Details

### 1. dynamo-9ae5027-systems-infrastructure-and-operations (Entitlement Audit)
- **Status:** Open (PR #1)
- **Soundness Issues Addressed:**
  - **Secrets Exposure (E4)**: Deleted `expected_audit_report.json` and hardcoded the metrics directly in `test_outputs.py`.
  - **Symlink Check (E5)**: Added `assert not os.path.islink(REPORT)` inside `_load_report()`.
  - **Discoverability Gap (B5)**: Explicitly documented the grant rule search family (modular multiplicative inverse parity mod small primes, base-specific digitsum parity, popcount, single base rules, XOR couplings) in `instruction.md`.
- **Plan File:** [dynamo-9ae5027-implementation-plan.md](file:///C:/Users/HP/Downloads/dynamo-9ae5027-implementation-plan.md)

### 2. dynamo-02af1ac-data-processing-and-etl (Timestamp Validation)
- **Status:** Open (PR #3)
- **Soundness Issues Addressed:**
  - **Pass@2 Difficulty Failure**: Rebuilt the dataset using `generate_events.py` by increasing `NUM_SOURCES` to 95 and events per source to (300, 500) to yield 53,000+ total rows. Added high-density transition-window boundary rows (spring-forward gaps and fall-back ambiguous `fold=1` non-default entries) plus near-miss UTCs.
- **Plan File:** [dynamo-02af1ac-implementation-plan.md](file:///C:/Users/HP/Downloads/dynamo-02af1ac-implementation-plan.md)

### 3. dynamo-0a247ea-data-processing-and-etl (GPS Segment Classifier)
- **Status:** Open (PR #1)
- **Soundness Issues Addressed:**
  - **Symlink Check (E5)**: Added symlink checks (`islink()`) in `_read_segments()` and `_read_summary()` inside `test_outputs.py`.
  - **Discoverability (B5)**: Documented Haversine-based speed calculations, interval-based stop grouping rules, and segment-merging rules in `instruction.md` to prevent specification gaps.
- **Plan File:** [dynamo-0a247ea-implementation-plan.md](file:///C:/Users/HP/Downloads/dynamo-0a247ea-implementation-plan.md)

### 4. dynamo-1a8cbcf-model-training-and-ml-infrastructure (FSDP Checkpoint Resharding)
- **Status:** Closed/Passed (PR #3)
- **Soundness Issues Addressed:**
  - **Symlink Check (E5)**: Added `assert not os.path.islink(path)` inside `test_files_exist()` in `test_outputs.py` to prevent symbolic link bypasses.
- **Plan File:** [dynamo-1a8cbcf-implementation-plan.md](file:///C:/Users/HP/Downloads/dynamo-1a8cbcf-implementation-plan.md)

### 5. dynamo-005c675-systems-infrastructure-and-operations (DFS Tie-Break Solver)
- **Soundness Issues Addressed:**
  - **Specification & Grading**: Created a new trace scenario `topo-tie-break.trace` testing DFS sorting tie-breaking order, generated `topo-tie-break.json`, registered the case in `test_outputs.py`, and updated `instruction.md` with explicit DFS topological sorting details.
- **Plan File:** [dynamo-005c675-implementation-plan.md](file:///C:/Users/HP/Downloads/dynamo-005c675-implementation-plan.md) (previously generated)

### 6. dynamo-a83e4bc-machine-learning-and-ai (Point Cloud Reconstructor)
- **Soundness Issues Addressed:**
  - **Point Count Verification**: Hardened `test_min_reconstructed_point_count` to verify that at least 150 *distinct* points are reconstructed. Updated instruction with camera trajectory and accuracy bounds.
- **Plan File:** [dynamo-a83e4bc-implementation-plan.md](file:///C:/Users/HP/Downloads/dynamo-a83e4bc-implementation-plan.md) (previously generated)

---

## Part 3: Local Testing and Verification Procedures

To run validation checks locally across WSL, Linux, or Windows Docker environments:

### Method 1: Using Harbor CLI (Recommended)
From the root of each repository directory, execute:
```bash
harbor task test --task .
```

### Method 2: Directly via Docker
For any task, build the environment image and run the test suite:
1. **Build the container:**
   ```bash
   docker build -t task-image task/environment/
   ```
2. **Execute solution and run verifier:**
   ```bash
   docker run --rm -it \
     -v "$PWD/task/solution:/solution" \
     -v "$PWD/task/tests:/tests" \
     task-image bash -c "python3 /solution/solve.py && pytest /tests/test_outputs.py"
   ```
