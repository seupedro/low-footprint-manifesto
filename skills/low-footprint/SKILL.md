---
name: low-footprint
description: Write, review and refactor software for the smallest possible memory and energy footprint, following The Low Footprint Manifesto. Use when choosing a language or stack for a new app, tool or service; when writing or reviewing code that runs on laptops, phones or other battery-powered devices; when the user mentions RAM usage, battery drain, energy efficiency, performance, binary size or flaky networks; or when the user asks for "low-footprint", "lean" or "efficient" software.
---

# Low Footprint

Software development got cheap, but RAM didn't. Batteries don't last forever.
Every byte and every CPU wakeup your code costs is paid by the user's device.

Follow the guidelines below in order. Rules marked **MUST** are not optional.
When a guideline conflicts with a feature the user asked for, keep the
feature and report the trade-off with numbers. Never silently drop
functionality to make a metric look better.

## 1. Pick a native, compiled stack

- For new code, prefer **Rust**. On Apple platforms, Swift is also a good fit.
  C, C++ and Zig are acceptable when the project already uses them.
- Avoid interpreted or VM-heavy runtimes (TypeScript/JavaScript on Node,
  Python, Ruby, and similar) and bundled-browser apps (Electron). If a UI
  needs web tech, use the system webview (e.g. Tauri) instead of shipping
  a browser.
- **Existing project in another language:** do NOT rewrite it unasked. Apply
  the remaining guidelines inside the current stack, and propose a port of
  the hot path only when measurements show it pays off.
- Ship optimized builds. For Rust, use a release profile like:

  ```toml
  [profile.release]
  opt-level = 3        # or "s"/"z" when binary size matters more
  lto = true
  codegen-units = 1
  panic = "abort"
  strip = true
  ```

## 2. Save as much memory as possible

- Stream data instead of loading whole files, responses or tables into memory.
- Reuse buffers; avoid needless clones, copies and intermediate collections.
- Pick compact data structures; pre-size collections when the size is known.
- Load lazily and free early. Every cache MUST have a size limit.
- Keep dependencies few and small; check what each one adds to binary size
  and resident memory before adding it.

## 3. Keep the energy impact as low as possible

Be obsessed with energy efficiency. The cheapest work is the work not done.

- Never busy-wait or poll in a tight loop. Prefer events, notifications and
  blocking I/O; if you must poll, use long intervals with backoff.
- Batch work and network requests: every radio and CPU wakeup has a tail cost.
- Coalesce timers, let the CPU idle, stop background work when the app is
  not visible, and respect the OS low-power mode.
- Skip redundant work: cache results (with limits), avoid re-rendering,
  re-parsing and re-computing unchanged data.
- Avoid gratuitous animations, high refresh rates and GPU use without need.

## 4. Assume the network is intermittent, slow and unstable

- Every network call MUST have a timeout.
- Retry with exponential backoff and jitter; make requests idempotent so
  retries are safe.
- Work offline first: keep local state, queue writes, sync when back online.
- Make large transfers resumable; never throw away partial progress on a
  dropped connection.
- Compress payloads, send only what changed, and degrade gracefully instead
  of blocking the UI on the network.

## 5. Measure, compare before and after, repeat

No optimization claim without numbers. Before changing anything, record a
baseline; after the change, measure again under the same conditions.

| What | Linux | macOS / iOS | Android |
|---|---|---|---|
| Peak memory (RSS) | `/usr/bin/time -v <cmd>` | `/usr/bin/time -l <cmd>`, Instruments (Allocations) | `adb shell dumpsys meminfo <package>` |
| Wall time / CPU | `hyperfine --warmup 3 '<cmd>'` | `hyperfine --warmup 3 '<cmd>'` | Android Studio Profiler |
| Energy | `perf stat -e power/energy-pkg/ <cmd>` (Intel/AMD RAPL) | `sudo powermetrics --samplers tasks --show-process-energy`, Xcode Energy gauge | `adb shell dumpsys batterystats` |
| Binary size | `ls -l`, `cargo bloat` | `ls -l` | APK Analyzer |

- Run each measurement more than once and report the spread, not a single
  lucky run.
- Keep improving while the numbers keep moving, but stop before a change
  compromises a feature or makes the code unmaintainable.

## Report

End every task that touched performance-relevant code with a before/after
table of the metrics you measured:

| Metric | Before | After | Change |
|---|---|---|---|
| Peak RSS | | | |
| Time | | | |
| Energy | | | |
| Binary size | | | |

If something could not be measured on this machine (e.g. no RAPL counters,
no device attached), say so explicitly instead of estimating.
