# PR Soundness & Gating Plan — dynamo-1a8cbcf-model-training-and-ml-infrastructure

## Task Metadata
- **Task Name**: `dynamo/fsdp-checkpoint-resharding`
- **Subcategory**: `Distributed training`
- **Git User ID**: `venkatasai31-07`
- **Local Path**: `C:\Users\HP\.gemini\antigravity-ide\scratch\dynamo-1a8cbcf-model-training-and-ml-infrastructure`

---

## 1. Quality & Gating Analysis (Green-Pipeline & Playbook Alignment)

### The Stated Rule vs. Empirical Trap
- **The Crux (Dormant Trap)**: The FSDP1 source shards are split using an uneven balanced layout (reconstructed from raw shard lengths), whereas the 5-way target outputs must strictly follow the native FSDP1 `SHARDED_STATE_DICT` layout (uniform `ceil`-divided shards with trailing zero-padding).
- **Playbook Alignment**: Under the **"row that is not there" / padding elements** principle:
  - If the agent assumes FSDP1 padding for both input and output, the source slice step fails.
  - If the agent empirically infers the balanced split from the source shards and applies the same balanced splitting formula to the output, it will produce output shapes that miss exactly one zero-padding element on trailing ranks (near-miss failure).
  - The instructions explicitly state that the 5-way outputs must be valid FSDP1 `SHARDED_STATE_DICT` shards for `world_size=5`. This ensures the rule is **conceptually entailed** (fair) but hard to compute without implementing the standard FSDP1 padding recipe.

### Gating & Timeout Setup
- **Timeout Pinned**: Set `[agent].timeout_sec = 3600.0` (hard cap) in `task.toml` to ensure agents do not fail via early timeout-confound, while retaining the reasoning difficulty of the resharding logic.

---

## 2. Technical Soundness Fixes

### Symlink Validation (QC Security Gate)
- **Problem**: The verifier loads model checkpoint files `/app/output/unit_{i}_rank_{r}.pt` directly. A compromised agent could symlink these to the pristine source directory to bypass resharding.
- **Fix**: Added `assert not os.path.islink(path)` inside `test_files_exist()` in `task/tests/test_outputs.py` before loading checkpoints.

---

## 3. Step-by-Step Gating & Verification Checklist

### Phase 1: Local Environment Validation (WSL, Docker, Harbor)
To guarantee the task environment builds cleanly and the verifier functions as expected without local path assumptions:
1. **Docker Build:**
   ```bash
   docker build -t fsdp-reshard task/environment/
   ```
2. **Execute Solution & Verifier (WSL/Linux/Docker):**
   ```bash
   docker run --rm -it \
     -v "$PWD/task/solution:/solution" \
     -v "$PWD/task/tests:/tests" \
     fsdp-reshard bash -c "python3 /solution/solve.py && pytest /tests/test_outputs.py"
   ```
3. **Harbor Task Test:**
   ```bash
   harbor task test --task .
   ```

### Phase 2: PR Submission & Gating Sequence
1. **Commit & Push Soundness Fix**:
   Commit the updated `task/tests/test_outputs.py` (with symlink check) and push to the fork.
2. **Monitor Pass@2 Pre-check**:
   - Verify that the two runs of Terminus-2 finish inside the 3600s budget.
   - Inspect the `pass2-output` artifact on failure to confirm that failures are valid (correct structure but wrong by padding/balanced split) and not timeouts/agent errors.
3. **Automated Review (QC Gates)**:
   - Ensure the static review has zero blocking issues.
4. **Pass@5 Gating**:
   - The pipeline will automatically run 5 trials. The task passes if at least 3 trials fail cleanly (stumped by the `ceil` zero-padding rule) and the oracle solves the task with reward 1.0.
