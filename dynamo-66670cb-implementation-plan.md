# Implementation Plan — dynamo-66670cb-hardware-embedded-and-low-level-systems

## Task Metadata
- **Task Name**: `dynamo/firmware-health`
- **Subcategory**: `Embedded and firmware`
- **Current PR**: PR #1 (closed/to be updated/re-opened via new PR)
- **Git User ID**: `venkatasai31-07`
- **Local Path**: `C:\Users\HP\.gemini\antigravity-ide\scratch\dynamo-66670cb-hardware-embedded-and-low-level-systems`

---

## QC Review Soundness Findings & Proposed Fixes

### 1. Symlinked Output Path (E5)
- **Problem**: The verifier reads `/app/health.json` directly, which could allow a symlink bypass if the output path is symlinked.
- **Fix**: Update `test_output_exists` in `task/tests/test_outputs.py` to assert that `PRODUCED` is not a symlink:
  ```python
  assert not PRODUCED.is_symlink(), "/app/health.json must not be a symlink"
  ```

### 2. Undocumented Integrity Fingerprint Schema (A5 & B3)
- **Problem**: The integrity fingerprint for v2 layouts requires hashing a specific JSON schema with mapped keys (`c`, `t`, `v`, `w` for critical, tick, voltage_bad, wear_violation) and boolean-to-integer casts, which is undocumented.
- **Fix**: Document the v2 integrity payload format explicitly in `instruction.md`:
  - It must be a JSON object with:
    - `"alert"`: Degraded/Alert minutes (int)
    - `"cause"`: Unique fault code count (int)
    - `"salt"`: The device's integrity salt string
    - `"ticks"`: A list of objects for critical intervals sorted by tick:
      - `{"c": int(critical), "t": tick, "v": int(voltage_bad), "w": int(wear_violation)}`
    - `"wear"`: `wear_ratio_milli` value (or `0` if disabled)
  - Hash this payload as a minified JSON string (no whitespace, sorted keys) using SHA-256.

### 3. Ambiguous & Undocumented Watchdog Burst Rules (B1, B4 & B7)
- **Problem**: The grouping logic for watchdog bursts and `watchdog_burst_count` is not fully specified.
- **Fix**: Document the exact watchdog burst grouping algorithm in `instruction.md`:
  - Watchdog events are sorted by `(wdg_group, tick, event_id)` (default group name is `"default"`).
  - A new burst is started for a group if no previous event in that group exists, or if the time difference `tick - last_tick` exceeds `wdg_gap_ticks` (configured per-device or defaults to `2`).

### 4. Unstated Leak Deduplication Policy (B6)
- **Problem**: Per-device power cost (`power_cost_nj`) calculations depend on deduplicating leak events that share a `frame_key` but have differing `nj` values. The survivor selection policy is unstated.
- **Fix**: Document the deduplication policy in `instruction.md`:
  - Leak events are sorted by `(tick, event_id)`.
  - For duplicate events sharing the same deduplication key (prioritized by `fault_key`, then `frame_key`, then `event_id`), only the first chronological/ID occurrence is kept.
  - The accumulated `nj` values of survivors are aligned to a 128-nj boundary (e.g. `align(nj, 128) = ((nj + 127) // 128) * 128`).

### 5. Narrow Precedence Coverage (C3)
- **Problem**: Swap of `fault_key` and `frame_key` precedence in deduplication logic still passes verification.
- **Fix**: Add a dedicated test or validation device configuration in `expected_health.json` (or verify in the tests) to ensure that the precedence order `fault_key` > `frame_key` > `event_id` is strictly verified.

---

## Detailed Step-by-Step Checklist

### Phase 1: Local Branch Setup
1. Checkout the `pr1-head` branch of `dynamo-66670cb-hardware-embedded-and-low-level-systems`.
2. Create/switch to a new branch for the soundness fixes:
   `git checkout -b task/fix-health-soundness`

### Phase 2: Update instruction.md
1. Write the complete, comprehensive task description to `task/instruction.md` detailing:
   - Input format (`/app/devices.json`, `/opt/fw/dumps`).
   - Output format (`/app/health.json`).
   - Detailed specifications for:
     - Critical interval checks.
     - Fault deduplication and precedence rules (`fault_key` > `frame_key` > `event_id`).
     - Watchdog burst grouping logic.
     - Leak energy deduplication, 128-nj alignment, and `power_cost_nj` calculation.
     - `wear_ratio_milli` calculation.
     - `integrity_fingerprint` schemas for both v1 and v2 devices.

### Phase 3: Harden Verifier
1. Edit `task/tests/test_outputs.py` to add `assert not PRODUCED.is_symlink()` check.

### Phase 4: Local Testing and Verification
Verify that the task builds and the verifier passes using docker/harbor commands:
- **Build Docker Image**:
  `docker build -t health-task task/environment/`
- **Test using Harbor CLI**:
  `harbor task test --task .`
- **Directly using Docker**:
  `docker run --rm -it -v "$PWD/task/solution:/solution" -v "$PWD/task/tests:/tests" health-task bash -c "python3 /solution/health.py && pytest /tests/test_outputs.py"`

### Phase 5: Push and Open PR
1. Commit all changes to the branch.
2. Push to GitHub fork and open a new PR.
