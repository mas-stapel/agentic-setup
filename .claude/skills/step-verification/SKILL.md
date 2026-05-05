---
name: step-verification
description: >
  Verification honesty rules for implementation steps. Covers the boundary between
  automatable criteria (test suites, type checking, linting) and GUI-dependent criteria
  (frame rate, drag interaction, visual screenshot comparison) that cannot be verified
  in a headless environment. Activate this skill for any step whose acceptance criteria
  include performance metrics, visual parity checks, or interaction tests against a
  running desktop application.
---

# Step Verification — Automated vs. GUI-Dependent Criteria

The central rule: **only claim what you can prove with terminal output.** Never invent measurements for criteria that require a running desktop window.

---

## 1. The Two Categories

Every acceptance criterion in a step belongs to exactly one category:

| Category | Definition | Examples |
|----------|------------|---------|
| **Automatable** | Can be verified by running a command in the current process and reading stdout | Test suite pass/fail, type errors, lint warnings, parse time (Rust integration test), sort time (unit test with timer) |
| **GUI-dependent** | Requires a running desktop window, display server, or human visual inspection | Frame rate, drag-and-drop interaction, screenshot visual comparison, DevTools recordings, app launch smoke test |

When writing acceptance criteria (tech-lead) or reviewing a step (fullstack-dev), classify each criterion before proceeding. If a criterion is GUI-dependent and the environment is headless, use an automatable proxy instead (see §3).

---

## 2. The Three Valid Reporting States

Every test or verification claim in a step report must use one of these three states — no others are valid:

### CONFIRMED
You ran the command and have terminal output proving it.

```
Tests:
- 24 tests written
- Verification: CONFIRMED
  > cargo test -p my-core 2>&1 | tail -3
  > test result: ok. 15 passed; 0 failed; 0 ignored
  > npx vitest run --reporter=verbose 2>&1 | tail -3
  > Tests  195 passed (195)
```

### NOT RUN
You could not run the command. State the reason explicitly.

```
Tests:
- 8 tests written
- Verification: NOT RUN — Tauri binary crate requires system GUI libraries not
  present in this environment. Core library tests pass (see CONFIRMED block above).
  Full binary test requires a desktop build on the target platform.
```

### PARTIAL
Automated criteria confirmed with stdout; GUI-dependent criteria deferred.

```
Tests:
- 12 tests written
- Verification: PARTIAL
  Automated (CONFIRMED):
  > npx vitest run 2>&1 | tail -2
  > Tests  195 passed (195)
  GUI-dependent (deferred):
  - Frame rate ≥ 55fps — requires DevTools attached to running WebView; cannot verify headlessly
  - Visual parity against design screenshots — requires running app; cannot verify headlessly
```

**The fourth state — fabricating plausible-sounding numbers — is not valid.** Do not write "Parse time: 1.84s" without actual `Instant`-based test output. Do not write "58fps" without a DevTools recording. Do not write "Visual parity confirmed" without running the app.

---

## 3. Automatable Proxies for Common GUI Criteria

When a plan step contains a GUI-dependent criterion, use one of these proxies instead. The proxy is weaker than the original but honest about what it measures.

| GUI criterion (cannot automate) | Automatable proxy |
|----------------------------------|-------------------|
| "Parse time < Xs wall-clock from file drop" | Integration test in core library using `std::time::Instant` against a programmatically generated fixture — Rust timing requires no GUI |
| "Sort N items in < Ys" | Unit test calling the sort function directly with `performance.now()` (JS) or `std::time::Instant` (Rust) |
| "Scroll at ≥ Nfps" | Vitest perf test: measure time for virtualizer to call `scrollToIndex` on 1000 rows via `performance.now()` in jsdom — asserts < Xms per scroll, not fps, but is honest about the proxy |
| "Visual parity against screenshot" | axe-core zero violations + CSS custom-property token assertions — confirms structural and token correctness, not pixel parity; document the distinction |
| "No console errors during E2E flow" | Unit tests on each component with mocked IPC — if tests pass with no thrown errors, document as "unit-level clean; E2E console audit deferred" |

When using a proxy, always label it as a proxy in your report: "proxy for X — does not measure fps directly."

---

## 4. Trigger Keywords

Load this skill when a plan step's acceptance criteria or description contains any of these phrases:

- "fps" / "frame rate" / "frames per second"
- "DevTools" / "Performance panel" / "profiler"
- "visual parity" / "screenshot" / "pixel"
- "drag" / "file drop" / "drop zone" (interaction-level, not component-level)
- "measured from file drop" / "wall-clock" / "from the running app"
- "smoke test" / "launch" (when referring to a GUI app, not a server)

When these phrases appear, classify each criterion as automatable or GUI-dependent before writing any code or report.
