# STABILITY_REPORT.md

## AMINE PS4 ALL LITE — Stability Report

**Purpose:** improve reliability and reduce unnecessary failures across the firmware branches already present in this package.

> This document is a stability checklist and test plan. It does not claim that a firmware branch is more stable until it has been tested on real hardware.

## Branches detected in this package

| Branch | Main contents | Stability priority |
|---|---|---|
| 5.05 | exploit + payload set | High |
| 6.72 | exploit + loader + payload set | High |
| 7.00 | modular WebKit/Lapse/ROP + payloads | High |
| 9.00 | modular WebKit/Lapse/ROP + payloads | High |
| 13.00 | latest core/Poops/Lapse + patch set | High |
| 13.52 | jb/core/offsets + patch set | Critical |
| `css/latest` | shared/latest source tree | Critical |

## Stability rules

### 1. Version isolation
- A firmware-specific offset, patch, ROP chain, or kernel component must never be silently reused for another firmware.
- Select the firmware branch before loading version-dependent components.
- If a required component is missing, fail cleanly instead of guessing a nearby version.
- Keep 13.52 assets isolated from 13.50/13.02 assets unless compatibility is explicitly verified.

### 2. Payload handling
- Load only the payload required by the selected operation.
- Avoid loading multiple large payloads automatically at startup.
- Do not duplicate the same payload in memory when one instance is sufficient.
- Verify payload presence/size before attempting to send or execute it.
- After a failed attempt, clear temporary state before retrying.

### 3. Memory stability
- Prefer one reusable worker/chain over repeatedly creating workers.
- Release temporary ArrayBuffers, views and message handlers after a failed run.
- Avoid retaining references to previous exploit attempts.
- Keep the default startup path minimal; optional tools should be loaded on demand.
- Do not run debugging/FTP/RTE components automatically unless requested.

### 4. Retry policy
Use bounded retries instead of an infinite loop:

1. First attempt: normal initialization.
2. Second attempt: reset temporary state and retry.
3. Third attempt: reload the affected page/worker.
4. If still unsuccessful: stop and show a clear failure message.

This prevents repeated failed attempts from accumulating memory/state corruption.

### 5. Worker lifecycle
- A worker should have one clear owner.
- Terminate failed/stale workers before creating replacements.
- Remove event listeners when a worker is discarded.
- Prevent two exploit chains from running simultaneously.
- Use explicit completion/failure states rather than timing alone.

### 6. Cache integrity
- Cache only files belonging to the selected firmware branch.
- After changing firmware-specific assets, invalidate the old cache.
- Do not mix cached JavaScript, offsets or patches from different revisions.
- Keep a recovery path that can reload the non-cached version.

### 7. UI / startup
The startup page should:
- detect the firmware;
- select the matching branch;
- show `READY`, `RUNNING`, `RETRY`, or `FAILED`;
- disable duplicate start actions while a run is active;
- avoid starting optional payloads automatically.

## Firmware test matrix

For every supported branch, test at least:

- Cold browser launch
- First attempt
- Second attempt after clean retry
- Page reload
- Cache-enabled launch
- Cache-disabled/recovery launch
- Successful completion
- Intentional failure/retry
- Payload loading after successful initialization
- Repeated runs without browser restart

### Suggested scoring

`Stability % = successful completed runs / total valid runs × 100`

Record **at least 20 independent runs** per firmware before comparing branches. A single successful run is not enough to claim improved stability.

## Failure logging

Record:

- firmware version
- browser/cache state
- attempt number
- stage reached
- error text
- whether the page was reloaded
- whether a worker was restarted
- final result

Example:

```text
FW: 13.52
Cache: ON
Attempt: 2
Stage: initialization
Worker reset: YES
Result: FAIL
Reason: initialization timeout
```

## Priority for this package

### 13.52
Highest priority because the branch contains a dedicated `jb.js`, `core.js`, `ps4_offsets.js`, patch file, cache page and loader components. Keep all of these version-specific and prevent fallback to unrelated offsets.

### 13.00
Use the same isolation rule for `core.js`, `chain_poops.js`, `ps4_offsets.js`, `run_poops.html`, `run_lapse.html` and the versioned patch files.

### 9.00 / 7.00
Keep ROP, Lapse, offsets and memory modules tied to their own branch. Avoid cross-branch imports.

### 6.72 / 5.05
Keep the simpler payload/loader path. Avoid adding unnecessary startup components merely to increase functionality.

## Recommended architecture

```text
Firmware detection
       |
       v
Exact version selector
       |
       +---- matching offsets
       +---- matching exploit chain
       +---- matching patches
       +---- matching payload interface
       |
       v
Single active worker
       |
       v
Bounded retry controller
       |
       +---- success -> finish
       |
       +---- failure -> cleanup -> retry
       |
       v
Final status / recovery
```

## What should NOT be changed blindly

Do not modify firmware-specific:
- kernel offsets
- ROP gadget addresses
- exploit primitives
- patch binaries
- payload binaries
- memory addresses

A change that looks like a stability improvement can make the branch fail completely if the value is firmware-specific.

## Acceptance criteria

A change should be considered a stability improvement only if it:

1. does not mix firmware-specific components;
2. does not increase duplicate workers or retained memory;
3. handles failure without requiring repeated manual reloads;
4. preserves the successful path;
5. improves the measured success rate over the previous build using the same test matrix.

## Final note

This report is intentionally conservative: it defines the changes that are safe to implement at the loader/UI/state-management level and the measurements needed to prove a real stability increase. It does **not** invent new offsets or claim unsupported firmware compatibility.
