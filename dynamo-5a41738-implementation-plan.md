# Implementation Plan — dynamo-5a41738-systems-infrastructure-and-operations

## Task Metadata
- **Task Name**: `dynamo/osprey-checkpoint-recover`
- **Subcategory**: `Storage and filesystem administration`
- **Current PR**: PR #1 (closed/to be updated/re-opened via new PR or branch update)
- **Git User ID**: `venkatasai31-07`
- **Local Path**: `C:\Users\HP\.gemini\antigravity-ide\scratch\dynamo-5a41738-systems-infrastructure-and-operations`

---

## QC Review Soundness Findings & Proposed Fixes

### 1. Narrow / Hardcodable Held-Out Coverage (Major Fix)
- **Problem**: The verifier only tests hidden cases that emit $\le 1$ output byte. A mutant emulator that incorrectly caches the `epoch_key` at the first `OUT` instruction (violating the requirement that it must be recomputed fresh every time) still passes all checks because no hidden test case triggers multiple `OUT` output bytes.
- **Fix**: Modify the hidden case `h_cycle` in `scripts/gen_fixtures.py` to run multiple `OUT` steps and generate multiple bytes of output. This will enforce that the `epoch_key` is correctly recomputed dynamically on every step.
  - Modify `h_cycle`'s `resume_tail` to execute two `OUT` instructions:
    ```
    PUSH 0x00050
    OUT
    PUSH 0x00060
    OUT
    HALT
    ```
  - Increase `resume_steps` for `h_cycle` from `3` to `5` to accommodate the extra instructions.
  - Run `python scripts/gen_fixtures.py` to regenerate all hidden programs, snapshots, and the `cases.json` manifest.

### 2. Oracle / Answers Readable by the Agent (Major Fix)
- **Problem**: The file `/app/sample.expected.json` contains the expected output of the sample program and is copied into the agent's container image via `Dockerfile`. This allows an agent to programmatically read/parse the answer file.
- **Fix**: Remove the expected answer file from the agent-visible container space.
  - Delete `COPY data/sample.expected.json /app/sample.expected.json` from `task/environment/Dockerfile`.
  - Remove `sample.expected.json` from git tracking and deleted/ignored files.
  - In `task/instruction.md`, replace references to `/app/sample.expected.json` with instructions on how the agent can run the local `/app/microvm` to generate the expected sample output themselves during development:
    - *Change*: "`/app/sample.expected.json` holds the exact JSON..."
    - *To*: "To see the expected output for the sample case, you can run `/app/microvm resume /app/sample.prog /app/sample.snap <steps>` during development."
  - In `task/tests/test_outputs.py`, hardcode the expected outputs for the sample test case directly in Python, since the test files are not copied to the agent's container and are only overlaid at verification time:
    ```python
    SAMPLE_EXPECTED_DATA = {
        "steps": 7,
        "registers": [5, 7, 11, 1, 10, 16, 0, 0],
        "output": "",
        "halted": True
    }
    ```

---

## Detailed Step-by-Step Checklist

### Phase 1: Local Branch & Setup
1. Checkout the `pr1-head` branch (which contains the `osprey-checkpoint-recover` task code).
2. Create/switch to a new branch for the submission update:
   `git checkout -b task/fix-soundness`

### Phase 2: Modify scripts/gen_fixtures.py & Regenerate
1. Open [gen_fixtures.py](file:///C:/Users/HP/.gemini/antigravity-ide/scratch/dynamo-5a41738-systems-infrastructure-and-operations/scripts/gen_fixtures.py):
   - Modify `h_cycle` block (lines 120-125) to add a second `OUT` sequence.
   - Set `resume_steps` for `h_cycle` to `5`.
2. Run fixture generation:
   `python scripts/gen_fixtures.py`
3. Verify that the regenerated hidden files under `task/tests/hidden/` are updated.

### Phase 3: Update Dockerfile & Instruction
1. Open [Dockerfile](file:///C:/Users/HP/.gemini/antigravity-ide/scratch/dynamo-5a41738-systems-infrastructure-and-operations/task/environment/Dockerfile):
   - Remove line 20: `COPY data/sample.expected.json /app/sample.expected.json`
2. Open [instruction.md](file:///C:/Users/HP/.gemini/antigravity-ide/scratch/dynamo-5a41738-systems-infrastructure-and-operations/task/instruction.md):
   - Rewrite line 3 to remove `/app/sample.expected.json` and guide the user to run `/app/microvm` to see sample output.
3. Delete [sample.expected.json](file:///C:/Users/HP/.gemini/antigravity-ide/scratch/dynamo-5a41738-systems-infrastructure-and-operations/task/environment/data/sample.expected.json) from disk.

### Phase 4: Hardcode Expected Output in Verifier
1. Open [test_outputs.py](file:///C:/Users/HP/.gemini/antigravity-ide/scratch/dynamo-5a41738-systems-infrastructure-and-operations/task/tests/test_outputs.py):
   - Remove `SAMPLE_EXPECTED` file path load.
   - Hardcode the expected JSON structure inside `test_sample_checkpoint_recovered`.

### Phase 4.5: Local Testing and Verification
1. Build the Docker environment image locally to verify the build process:
   `docker build -t osprey-task task/environment/`
2. Test the oracle solution and verification scripts locally via `harbor` (if installed) or directly in a container (Linux / WSL / Docker Desktop):
   - **Using Harbor CLI**:
     `harbor task test --task .`
   - **Directly using Docker** (to ensure the verifier rejects wrong implementations and accepts the oracle):
     `docker run --rm -it -v "$PWD/task/solution:/solution" -v "$PWD/task/tests:/tests" osprey-task bash -c "bash /solution/solve.sh && pytest /tests/test_outputs.py"`

### Phase 5: Push and Open PR
1. Commit all changes to the `task/fix-soundness` branch.
2. Push to GitHub fork and open a PR.

