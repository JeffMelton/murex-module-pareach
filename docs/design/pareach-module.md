# Murex `pareach` Module — Design Document

## Overview

`pareach` is a Murex module that provides ergonomic, high‑performance parallel iteration for ad‑hoc shell workflows. It wraps an external parallel runner (preferred: `rust-parallel`) to execute a user‑supplied Murex block concurrently over a sequence of items while preserving Murex idioms (typed pipelines, blocks, `$.` meta, variable binding).

Key points:
- No fallback to the built‑in `foreach --parallel` (performance constraints). `pareach` always uses an external engine or fails fast.
- Align feature set with `rust-parallel` for: ordered output, collection vs streaming, concurrency (jobs), per‑task timeout, exit on error, progress bar.
- Keep UX idiomatic to Murex and comparable to Elvish `peach`, Nushell `par-each`, and PowerShell `ForEach-Object -Parallel`.

Non-goals:
- Background jobs/FG switching (Murex job control is limited; not in scope).
- Re-architecting Murex core concurrency.


## User Experience

Two invocation styles, mirroring `foreach` for familiarity:

1) Variable binding

```
<stdin-seq> -> pareach item { out $item }
```

2) Stdin block

```
<stdin-seq> -> pareach { -> some-command }
```

Flags (first-pass set, mapped to `rust-parallel` options where possible):
- `-n, --concurrency <int>`: number of workers (maps to `jobs`). `0` or negative = unbounded.
- `--ordered`: preserve input order (maps to `keep order`).
- `--collect`: gather outputs and emit once on completion (vs streaming).
- `--timeout <dur>`: per‑task timeout (pass‑through). Examples: `500ms`, `5s`, `2m`.
- `--halt-on-error`: stop on first error (maps to `exit on error`). Default is continue‑on‑error.
- `--progress`: enable a progress bar (stderr).
- `--channel-capacity <int>` (advanced): override internal channel/queue capacity used by the engine; default is heuristic. See “Capacity & Backpressure”.
- `--using var1,var2,...`: capture named Murex variables by value and make them available to the per‑item block.

Notes:
- No background/daemon mode.
- If `rust-parallel` is not available, `pareach` errors with a clear message and help to install via `murex-package`.

Examples:
- Double numbers with eight workers and ordered output:

```
ja [1..100] -> pareach --concurrency 8 --ordered i { out ($i * 2) } -> sum
```

- Process files concurrently, streaming results, stop on first failure:

```
ls -1 | pareach -n 6 --halt-on-error { -> open -> mime -> head }
```

- HTTP fetch with per‑task timeout and progress:

```
open urls.json -> pareach --timeout 10s --progress u { http get $u -> status }
```


## Data Model and Typing

Input iteration rules:
- If stdin is a sequence type (JSON array, table/list), iterate elements.
- If stdin is a map/jmap and a key/value form is needed, this is a phase‑2 enhancement (see “Future Work”).
- If stdin is plain text, iterate by lines.

Per iteration, two bindings are supported:
- Variable form: bind `$item` (name chosen by user) to the element’s value; set `$.` meta to `{ "i": <index> }`. The block reads from `$item` and can also read stdin if the block uses `->` style.
- Stdin form: set the element as the block’s stdin (typed when possible) and set meta `$.i` to the index. Use `$using` for captured variables.

Typing:
- Internally, the orchestrator serializes items to JSON for interprocess communication with the external engine and child Murex processes.
- Output defaults:
  - Streaming: emit each item’s result as it completes. For JSON input, default to `jsonl`; otherwise preserve the detected input type if feasible. Casting by the user remains straightforward (`-> cast yaml`, etc.).
  - `--collect`: accumulate into an array and emit once (JSON array when input is JSON or when streaming implied `jsonl`).
- We capture the input type to set reasonable defaults; exact preservation across all formats is not guaranteed (Murex offers explicit `cast` when needed).


## Architecture

High-level components:
- Orchestrator (Murex function in a module):
  - Reads stdin into an item stream (without loading entire input when streaming).
  - Normalizes each item into a JSON envelope: `{ index, item, using }`.
  - Spawns the external engine (`rust-parallel`) with configured flags.
  - Feeds items to the engine’s input; receives per‑task outputs.
  - Aggregates results: preserves order when `--ordered`, buffers in `--collect`.
  - Emits progress to stderr when `--progress`.
  - Handles timeouts and error modes (halt/continue) in alignment with the engine.

- Worker wrapper (per task):
  - Launch a Murex child executing the user block. Two wiring options depending on engine capability:
    1) Stdin mode: engine feeds JSON envelope via stdin to the worker. The worker decodes to `$item` and `$using.*`, sets `$.` meta, then runs the block.
    2) Arg placeholder mode: engine substitutes a placeholder with a base64-encoded JSON envelope; the wrapper decodes it before running the block. (Used if stdin mode is not supported.)
  - The worker’s stdout is the task result; stderr is surfaced to the parent stderr with context when possible.

Engine invocation:
- `rust-parallel` is executed with flags derived from the user options: `--jobs`, `--keep-order`, `--timeout`, `--exit-on-error`, and a progress option. The exact CLI mapping follows the engine’s manual.
- The orchestrator composes a command that runs Murex for each item, e.g.: `murex -c '<worker-preamble>; { user-block }'` with proper quoting, or uses a temporary script file referenced by path to avoid quoting complexity.

Cancellation:
- SIGINT/SIGTERM propagate to the engine and transitively to workers. In `--halt-on-error` mode, the orchestrator drains or stops according to the engine semantics, then exits non‑zero.


## Error Handling Semantics

- Continue-on-error (default):
  - Capture per‑item errors, emit to stderr with `{ index, error, item-summary }` context, continue processing pending items.
  - Exit code `0` if all succeeded; non‑zero if any failed and `--strict-exit` (future option) is enabled.

- Halt-on-error (`--halt-on-error`):
  - Stop scheduling new tasks when the first error occurs; request engine cancellation of in‑flight workers where supported.
  - Ordered mode flushes any already completed items preceding the failure index; collect mode returns partial results only if explicitly allowed (future `--collect=best-effort`).
  - Exit with non‑zero code.


## Capacity & Backpressure

- Default strategy: set engine channel/queue capacity to `max(jobs * 2, 1024)` to balance throughput and memory. This offers headroom for fast producers and slow consumers.
- Expose `--channel-capacity` for advanced tuning. Document trade‑offs:
  - Larger capacity reduces producer stalls but increases memory and end‑to‑end latency jitter.
  - Smaller capacity increases backpressure, improving responsiveness and limiting memory.


## Capturing Variables (`--using`)

- Users can pass named Murex vars (including `jmap`/JSON values) into the worker closure:

```
let auth = ({ token: "..." })
open urls.json -> pareach --using auth u {
    http get $u --header ("Authorization: Bearer $(auth.token)")
}
```

Implementation:
- Orchestrator snapshots the named variables, encodes as JSON under `using` in the item envelope.
- Worker preamble restores them as `$using.<name>` (or rebinds to `$<name>` if desirable). Prefer a `$using` namespace to avoid accidental overwrite.


## Output Aggregation

- Streaming (default): write each completed result to stdout as soon as available.
- `--ordered`: buffer results by `index`; flush sequentially to preserve input order.
- `--collect`: gather into an array and emit at the end (JSON array if input is JSON/`jsonl`), with memory usage proportional to result size.


## Cross-Platform & Packaging

- OS support: Linux, macOS, Windows. Quoting strategies:
  - Prefer temporary script files for the worker to avoid complex quoting on Windows.
  - Use platform‑aware path handling.
- Dependency:
  - Detect `rust-parallel` at runtime. If missing, print install guidance.
  - Provide a `murex-package` package that can fetch prebuilt binaries per OS/arch or build from source where feasible.
- Version checks: ensure minimum compatible `rust-parallel` version; error with actionable message if older.


## Security Considerations

- Inputs and `$using` data are user‑controlled; treat as untrusted and avoid shell interpolation in the engine command. Prefer stdin or base64‑encoded JSON for item data.
- Limit temp file permissions where used (0600), and delete after use.


## Testing Strategy (Behavioural `.mx`)

- Correctness:
  - Streaming vs `--collect` outputs.
  - `--ordered` preserves order under mixed task durations.
  - `--timeout` cancels long tasks per item.
  - `--halt-on-error` stops early; default continues.
  - `$using` variables visible and correct inside the block.
  - Basic type preservation for JSON inputs (json → jsonl / collect → json array).

- Robustness:
  - Large inputs with bounded memory (streaming; limited channel capacity).
  - SIGINT cancellation.

- Performance sanity:
  - Demonstrate throughput vs single‑threaded baseline (documentation note; avoid heavy CI loads).


## Implementation Sketch (Pseudocode)

Worker preamble (executed inside each task):

```
# stdin carries a JSON envelope: { index, item, using }
json read -> foreach env {  # single object
    $. = ({ i: $.index })
    let using = $.using     # namespace for captured vars
    let item  = $.item
}

{ user-block }  # sees $item and $using.*; can also read from stdin depending on mode
```

Engine wiring (conceptual):

```
# Orchestrator
fn pareach (name?: str "variable binding" , ...flags...) {
  # 1) Parse flags; detect rust-parallel; build engine args
  # 2) Build worker script (tmp file) that:
  #    - decodes envelope → sets $.i, $using, and either $name or stdin
  #    - runs the user block
  # 3) Read stdin incrementally → for each item produce JSON envelope line
  # 4) Pipe envelopes to rust-parallel stdin
  # 5) Read per-task outputs; aggregate per --ordered/--collect; forward to stdout
  # 6) Handle timeout, exit-on-error, progress per engine behavior
}
```


## Performance Considerations

- Process per item: higher overhead than in‑process goroutines, but acceptable for ad‑hoc shell usage and IO‑bound tasks. Provides superior parallelism vs current built‑in.
- Memory:
  - Streaming keeps memory bounded (buffers per in‑flight task and small queues).
  - `--collect` scales with output size; document expectations.
- Channel capacity affects latency/throughput; default heuristic and advanced override via `--channel-capacity`.


## Open Questions & Future Work

- Retry policy (exponential backoff per item) — optional future flag.
- Parallel key/value (`--jmap` two‑block form) — phase‑2.
- Typed IPC to child Murex via `EnvMethod`/`EnvDataType` — phase‑2; would reduce JSON encode/decode.
- Richer progress integration (percent from total item count; ETA) and customization.
- Enhanced error reporting channel (structured jsonl on stderr).
- Windows‑specific quoting/encoding refinements; temp‑file worker default.


## Deliverables (MVP)

1) Module package (`murex-module-pareach`):
   - Function `pareach` with flags: `--concurrency/-n`, `--ordered`, `--collect`, `--timeout`, `--halt-on-error`, `--progress`, `--channel-capacity`, `--using`.
   - Runtime detection of `rust-parallel` and version.
   - Worker script generation and safe invocation.
2) Behavioural tests under `behavioural/pareach_*.mx` covering ordering, timeout, errors, `$using`, streaming vs collect.
3) Documentation under the module repo with examples and caveats; cross‑reference in Murex docs.
