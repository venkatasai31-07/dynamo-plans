# Soundness & Gating Plan — dynamo-a177a23-debugging-and-repair

## Task Metadata
- **Task Name**: `dynamo/mux-decoder-repair`
- **Subcategory**: `Runtime Bug repair`
- **Current PR**: PR #2 (open)
- **Git User ID**: `venkatasai31-07`
- **Local Path**: `C:\Users\HP\.gemini\antigravity-ide\scratch\dynamo-a177a23-debugging-and-repair`

---

## QC Review Soundness Findings & Proposed Fixes

### 1. Nondeterminism in Generation, Solution, or Verification (QC Major)
- **Problem**: The verifier uses unseedable cryptographically secure random values via `secrets.token_hex(16)` to generate scenario sentinels and runner marker strings, causing non-deterministic test outputs across runs.
- **Fix**: Replaced `secrets` with a seeded random generator helper `_token_hex(16)` using a fixed seed (`random.Random(1337)`) inside `task/tests/test_outputs.py`.

### 2. Reward / Harness Plumbing Exploit (QC Major)
- **Problem**: The verifier executes agent-writable code (the multiplexed decoder implementation in `/app/muxcodec/decoder.py`) in subprocesses running under the root user. A malicious agent could modify the decoder code to write `1` to `/logs/verifier/reward.txt` or execute arbitrary system-level commands.
- **Fix**: Hardened the harness subprocess execution inside `task/tests/test_outputs.py`. On non-Windows platforms, subprocesses run de-privileged as the `nobody` user. Permissions on the temporary directories and copied package files are set dynamically (`0o755` for directories, `0o644` for files) to ensure readability and execution under the de-privileged user without compromising the host filesystem.

---

## Detailed Step-by-Step Gating & Verification Checklist

### Phase 1: Local Environment Validation (WSL, Docker, Harbor)
To guarantee the task environment builds cleanly and the verifier functions as expected without local path assumptions:
1. **Docker Build:**
   ```bash
   docker build -t mux-decoder-task task/environment/
   ```
2. **Execute Solution & Verifier (WSL/Linux/Docker):**
   ```bash
   docker run --rm -it \
     -v "$PWD/task/solution:/solution" \
     -v "$PWD/task/tests:/tests" \
     mux-decoder-task bash -c "python3 /solution/solve.py && pytest /tests/test_outputs.py"
   ```
3. **Harbor Task Test:**
   ```bash
   harbor task test --task .
   ```

### Phase 2: Git Submission & PR Updates
1. Commit all files including the newly added `soundness-gating-plan.md`.
2. Push to branch `pr2-head` (or current PR branch) on the GitHub fork of `venkatasai31-07`.
3. Check the GitHub Actions pre-check (pass@2) comment.
